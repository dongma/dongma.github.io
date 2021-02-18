---
layout: post
title: spark standalone模式启动源码分析
---
## spark standalone模式启动源码分析

> spark目前支持以standalone、Mesos、YARN、Kubernetes等方式部署，本文主要分析apache spark在standalone模式下资源的初始化、用户application的提交，在spark-submit脚本提交应用时，如何将--extraClassPath等参数传递给Driver等相关流程。

从`spark-submit.sh`提交用户`app`开始进行分析，`--class` 为`jar`包中的`main`类，`/path/to/examples.jar`为用户自定义的`jar`包、`1000`为运行`SparkPi`所需要的参数。

```shell
# Run on a Spark standalone cluster in client deploy mode
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000
```

在`spark`的`bin`目录下的`spark-submit.sh`脚本中存在调用`spark-class.sh`，同时会将`spark-submit`的参数作为`"$@"`进行传递：

```shell
# 在用spark-submit提交程序jar及相应参数时，调用该脚本程序  "$@"为执行脚本的参数，将其传递给spark-class.sh
exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```

在`spark-class.sh`中会将参数传递给`org.apache.spark.launcher.Main`用于启动程序：

<!-- more -->

```shell
# The exit code of the launcher is appended to the output, so the parent shell removes it from the
# command array and checks the value to see if the launcher succeeded.
build_command() {
  "$RUNNER" -Xmx128m -cp "$LAUNCH_CLASSPATH" org.apache.spark.launcher.Main "$@"
  printf "%d\0" $?
}

# Turn off posix mode since it does not allow process substitution
set +o posix
CMD=()
while IFS= read -d '' -r ARG; do
  CMD+=("$ARG")
# 调用build_command()函数将参数传递给 org.apache.spark.launcher.Main这个类，用于启动用户程序
done < <(build_command "$@")
```

参数传递到``org.apache.spark.launcher.Main#main(String[] argsArray)`方法用于触发运行`spark`应用程序，当`class`为`SparkSubmit`时，从`args`中解析校验请求参数，校验参数、加载`classpath`中的`jar`、向`executor`申请的资源来构建`bash`脚本，触发`spark`执行应用程序。

```scala
public static void main(String[] argsArray) throws Exception {  
		/* 通过spark-submit脚本启动时为此形式，exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@" */
    if (className.equals("org.apache.spark.deploy.SparkSubmit")) {
      AbstractCommandBuilder builder = new SparkSubmitCommandBuilder(args);
      /* 从spark-submit.sh中解析请求参数，获取spark参数构建执行命令 AbstractCommandBuilder#buildCommand */
      cmd = buildCommand(builder, env, printLaunchCommand);
    } else {
      AbstractCommandBuilder builder = new SparkClassCommandBuilder(className, args);
      cmd = buildCommand(builder, env, printLaunchCommand);
    }
  	  /*
       * /usr/latest/bin/java -cp [classpath options] org.apache.spark.deploy.SparkSubmit --master yarn-cluster
       * --num-executors 100 --executor-memory 6G --executor-cores 4 --driver-memory 1G --conf spark.default.parallelism=1000
       *  --conf spark.storage.memoryFraction=0.5 --conf spark.shuffle.memoryFraction=0.3
       * */
      for (String c : bashCmd) {
        System.out.print(c);
        System.out.print('\0');
      }
}
```

进入`org.apache.spark.deploy#main()`方法体，`parseArguments(args)`方法会解析`spark-submit.class`的参数、加载系统环境变量（`ignore spark`无关的参数），会调用父类 `SparkSubmitOptionParser#parse(List<String> args)`方法解析参数，然后通过`handle()`、`handleUnknown()`、`handleExtraArgs()`获得应用程序需要的`jar`（`--jars`参数）和参数。

```scala
def doSubmit(args: Array[String]): Unit = {
  // Initialize logging if it hasn't been done yet. Keep track of whether logging needs to
  // be reset before the application starts.
  val uninitLog = initializeLogIfNecessary(true, silent = true)

  val appArgs = parseArguments(args)
  if (appArgs.verbose) {
    logInfo(appArgs.toString)
  }
  appArgs.action match {
    case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)
    case SparkSubmitAction.KILL => kill(appArgs)
    case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
    case SparkSubmitAction.PRINT_VERSION => printVersion()
  }
}
```
在提交应用未指定`action`参数时，默认为`submit`类型，以下为`SparkSubmitArguments#loadEnvironmentArguments()`解析的内容
```scala
// Action should be SUBMIT unless otherwise specified
action = Option(action).getOrElse(SUBMIT)
```

继续跟踪到`SparkSubmit#submit(SparkSubmitArguments, Boolean)`方法，在`spark 1.3`以后逐渐采用`rest`协议进行数据通信，直接进入`doRunMain(args: SparkSubmitArguments, uninitLog: Boolean)`方法，调用`prepareSubmitEnvironment(args)`解析应用程序参数，通过自定义类加载器`MutableURLClassLoader`下载`jar`包加载`class`进入`jvm`：

```scala
private def runMain(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
    val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)
    /* 设置当前线程的classLoader，MutableURLClassLoader实现了URLClassLoader接口，用于自定义类的加载 */
    val loader =
      if (sparkConf.get(DRIVER_USER_CLASS_PATH_FIRST)) {
        new ChildFirstURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      } else {
        new MutableURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      }
    /* 线程默认类加载器假如不设置 采用的是系统类加载器，线程上下文加载器会继承其父类加载器 */
    Thread.currentThread.setContextClassLoader(loader)

    /* 只有在yarn client模式下，用户的jar、通过--jars上传的jar全部被打包到loader的classpath里面.所以说，只要不少包 无论隐式
     * 引用其它包的类还是显式的引用，都会被找到.
     * --jars 参数指定的jars在yarn cluster模式下，直接是被封装到childArgs里面了，传递给了yarn.client
     */
    for (jar <- childClasspath) {
      addJarToClasspath(jar, loader)
    }

    var mainClass: Class[_] = null
    try {
      /* 采用的是上面的类加载器用于加载类class */
      mainClass = Utils.classForName(childMainClass)
    } catch {
      case e: ClassNotFoundException =>
        logWarning(s"Failed to load $childMainClass.", e)
    }

    val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
      mainClass.newInstance().asInstanceOf[SparkApplication]
    } else {
      // SPARK-4170
      if (classOf[scala.App].isAssignableFrom(mainClass)) {
        logWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
      }
      new JavaMainApplication(mainClass)
    }

    try {
      /* standalone模式在 SparkSubmit#prepareSubmitEnvironment(args)中将childMainClass设置为RestSubmissionClient */
      app.start(childArgs.toArray, sparkConf)
    } catch {
      case t: Throwable =>
        throw findCause(t)
    }
  }
}
```



