---
layout: post
title: spark standalone模式启动源码分析
---
## spark standalone模式启动源码分析

> spark目前支持以standalone、Mesos、YARN、Kubernetes等方式部署，本文主要分析apache spark在standalone模式下资源的初始化、用户application的提交，在spark-submit脚本提交应用时，如何将--extraClassPath等参数传递给Driver等相关流程。

从`spark-submit.sh`提交用户`app`开始进行分析，`--class` 为`jar`包中的`main`类，`/path/to/examples.jar`为用户自定义的`jar`包、`1000`为运行`SparkPi`所需要的参数（基于`spark 2.4.5`分析）。

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

参数传递到`org.apache.spark.launcher.Main#main(String[] argsArray)`方法用于触发运行`spark`应用程序，当`class`为`SparkSubmit`时，从`args`中解析校验请求参数，校验参数、加载`classpath`中的`jar`、向`executor`申请的资源来构建`bash`脚本，触发`spark`执行应用程序。

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

在之前调用`prepareSubmitEnvironment(args)`时已将`mainClass`实例化为`RestSubmissionClient`，使用`app.start(childArgs.toArray, sparkConf)`使用`restclient`提交请求，在`RestSubmissionClient.filterSystemEnvironment(sys.env)`方法会过滤掉非`SPARK_`或`MESOS_`开头的环境变量。

```scala
override def start(args: Array[String], conf: SparkConf): Unit = {
  if (args.length < 2) {
    sys.error("Usage: RestSubmissionClient [app resource] [main class] [app args*]")
    sys.exit(1)
  }
  val appResource = args(0)
  val mainClass = args(1)
  val appArgs = args.slice(2, args.length)  /* 参数的顺序是(args.primaryResource(用户jar), args.mainClass, args.childArgs) */
  // 过滤系统中的环境变量，只保留以 SPARK_ or MESOS_开头的环境变量
  val env = RestSubmissionClient.filterSystemEnvironment(sys.env)
  run(appResource, mainClass, appArgs, conf, env)
}
```

追踪到`RestSubmissionClientApp#run`方法，将`sparkConf`转换为`sparkProperties`并进行过滤（只保留`spark.`开头的属性），继续跟踪`client.createSubmission(submitRequest)`提交`rest`请求。

```scala
/** Submits a request to run the application and return the response. Visible for testing. */
def run(
  appResource: String,
  mainClass: String,
  appArgs: Array[String],
  conf: SparkConf,
  env: Map[String, String] = Map()): SubmitRestProtocolResponse = {
  val master = conf.getOption("spark.master").getOrElse {
    throw new IllegalArgumentException("'spark.master' must be set.")
  }
  /* SparkConf创建的时候获取的配置 (以spark.开头的), 转换为SparkProperties */
  val sparkProperties = conf.getAll.toMap
  val client = new RestSubmissionClient(master)
  val submitRequest = client.constructSubmitRequest(
    appResource, mainClass, appArgs, sparkProperties, env)
  /* 发送创建好的消息Message(submitRequest)到Driver端, postJson(url, request.toJson)解析rest返回的结果 */
  client.createSubmission(submitRequest)
}
```

在`RestSubmissionClientApp#createSubmission()`方法中验证所有`masters`地址，开始构建`submitUrl`然后逐个向`master`发送请求。在每次发送请求时都会验证`master`是否可用，当不可用时会将其添加到`lostMasters`列表中。至此，在`standalone`模式下提交一个`spark application`的流程就到此为止。

```scala
/**
 * Submit an application specified by the parameters in the provided request.
 * If the submission was successful, poll the status of the submission and report
 * it to the user. Otherwise, report the error message provided by the server.
 */
def createSubmission(request: CreateSubmissionRequest): SubmitRestProtocolResponse = {
  logInfo(s"Submitting a request to launch an application in $master.")
  var handled: Boolean = false
  var response: SubmitRestProtocolResponse = null
  for (m <- masters if !handled) {
    validateMaster(m)
    val url = getSubmitUrl(m)
    response = postJson(url, request.toJson)
    response match {
      case s: CreateSubmissionResponse =>
      if (s.success) {
        reportSubmissionStatus(s)
        handleRestResponse(s)
        handled = true
      }
      case unexpected =>
      handleUnexpectedRestResponse(unexpected)
    }
  }
  response
}
```

客户端提交应用的部分看完了，现在来分析`master`端如何接收请求并进行处理，在`start-master.sh`脚本中存在以下脚本，可以以`org.apache.spark.deploy.master.Master`作为分析代码的入口。

```sh
# NOTE: This exact class name is matched downstream by SparkSubmit.
# Any changes need to be reflected there.
CLASS="org.apache.spark.deploy.master.Master"
if [ "$SPARK_MASTER_WEBUI_PORT" = "" ]; then
  SPARK_MASTER_WEBUI_PORT=8080
fi

"${SPARK_HOME}/sbin"/spark-daemon.sh start $CLASS 1 \
  --host $SPARK_MASTER_HOST --port $SPARK_MASTER_PORT --webui-port $SPARK_MASTER_WEBUI_PORT \
  $ORIGINAL_ARGS
```

在`Master#main`方法中启动了`RPC`运行环境以及`Endpoint`，`RpcEndpoint`：`RPC`端点 ，`Spark`针对于每个节点（`Client/Master/Worker`）都称之一个`Rpc`端点，且都实现`RpcEndpoint`接口，内部根据不同端点的需求，设计不同的消息和不同的业务处理，如果需要发送（询问）则调用`Dispatcher`。

```scala
def main(argStrings: Array[String]) {
  Thread.setDefaultUncaughtExceptionHandler(new SparkUncaughtExceptionHandler(
    exitOnUncaughtException = false))
  Utils.initDaemon(log)
  val conf = new SparkConf
  val args = new MasterArguments(argStrings, conf)
  /* 创建rpc环境 和 Endpoint(供Rpc调用)，在Spark中 Driver， Master ，Worker角色都有各自的Endpoint，相当于各自的Inbox */
  val (rpcEnv, _, _) = startRpcEnvAndEndpoint(args.host, args.port, args.webUiPort, conf)
  rpcEnv.awaitTermination()
}
```

`Master`继承了`ThreadSafeRpcEndpoint`类，重写的`receive`方法用于接收`netty`提交的请求，这部分为`Master`服务启动的过程。

```scala
override def receive: PartialFunction[Any, Unit] = {
  /* 在AppClient向master注册Application后才会触发master的schedule函数进行launchExecutors操作 */
  case RegisterApplication(description, driver) =>
  // TODO Prevent repeated registrations from some driver
  if (state == RecoveryState.STANDBY) {
    // ignore, don't send response
  } else {
    logInfo("Registering app " + description.name)
    val app = createApplication(description, driver)
    registerApplication(app)
    logInfo("Registered app " + description.name + " with ID " + app.id)
    persistenceEngine.addApplication(app)
    driver.send(RegisteredApplication(app.id, self))
    schedule()  /* todo: 用于调度Driver，具体的调度内容需要详细的看 */
  }
}
```

`RestSubmissionClient`提交的请求统一由`StandaloneRestServer#handleSubmit(String, SubmitRestProtocolMessage, HttpServletResponse)`统一进行处理，通过`case CreateSubmissionRequest`表达式匹配请求的类型，使用`DeployMessages.RequestSubmitDriver(driverDescription)`申请启动`Driver`。

```scala
// A server that responds to requests submitted by the [[RestSubmissionClient]].
// This is intended to be embedded in the standalone Master and used in cluster mode only.
protected override def handleSubmit(
  requestMessageJson: String,
  requestMessage: SubmitRestProtocolMessage,
  responseServlet: HttpServletResponse): SubmitRestProtocolResponse = {
  requestMessage match {
    case submitRequest: CreateSubmissionRequest =>
    /* 构建好所有的参数DriverDescription，用于向Driver端发送请求 */
    val driverDescription = buildDriverDescription(submitRequest)
    /* Driver构建完成后正式向Master发起一个请求，向master请求资源 */
    val response = masterEndpoint.askSync[DeployMessages.SubmitDriverResponse](
      DeployMessages.RequestSubmitDriver(driverDescription))
    val submitResponse = new CreateSubmissionResponse
    submitResponse.serverSparkVersion = sparkVersion
    submitResponse.message = response.message
    submitResponse.success = response.success
    submitResponse.submissionId = response.driverId.orNull
    val unknownFields = findUnknownFields(requestMessageJson, requestMessage)
    if (unknownFields.nonEmpty) {
      // If there are fields that the server does not know about, warn the client
      submitResponse.unknownFields = unknownFields
    }
    submitResponse
  }
}
```

在`Master#receiveAndReply()`方法中用`createDriver(description)`对`DriverDescription`再进行一次封装，同时通过`schedule()`进行资源调度到`Worker`上（在`schedule`方法中调用`launchDriver`的方法，会向`Worker`发送一个`LaunchDriver`类型请求），最后`reply`进行`rest`请求响应。

```scala
override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
  case RequestSubmitDriver(description) =>
  if (state != RecoveryState.ALIVE) {
    val msg = s"${Utils.BACKUP_STANDALONE_MASTER_PREFIX}: $state. " +
    "Can only accept driver submissions in ALIVE state."
    context.reply(SubmitDriverResponse(self, false, None, msg))
  } else {
    logInfo("Driver submitted " + description.command.mainClass)
    val driver = createDriver(description)
    persistenceEngine.addDriver(driver)
    waitingDrivers += driver
    drivers.add(driver)
    schedule()  // 执行调度的逻辑schedule()

    // TODO: It might be good to instead have the submission client poll the master to determine
    //       the current status of the driver. For now it's simply "fire and forget".
    context.reply(SubmitDriverResponse(self, true, Some(driver.id),
                                       s"Driver successfully submitted as ${driver.id}"))
  }
}
```

将视角转到`Worker#receive()`方法中，通过模式匹配`case LaunchDriver(driverId, driverDesc)`进入如下代码，然后调用`driver.start()`启动程序。

```scala
case LaunchDriver(driverId, driverDesc) =>
	logInfo(s"Asked to launch driver $driverId")
  /*
   * 在RestSubmissionClient向StandaloneRestServer提交launchDriver请求后，实际上在StandaloneRestServer进行了一层封装
   * DriverWrapper. 所以，在此处启动的类是DriverWrapper 而不是用户程序本身，在该main方法里，主要是用自定义类加载器加载了用户的
   * main方法，然后开始启动用户程序 初始化sparkContext等;
   */
   val driver = new DriverRunner(
      conf,
      driverId,
      workDir,
      sparkHome,
      /*
       * 此处的Command就是在StandaloneRestServer封装好的
       * val command = new Command("org.apache.spark.deploy.worker.DriverWrapper", Seq("{{WORK_URL}}",
       *  "{{USER_JAR}}", mainClass)) ++ appArgs,   // args to the DriverWrapper
       */
      driverDesc.copy(command = Worker.maybeUpdateSSLSettings(driverDesc.command, conf)),
      self,
      workerUri,
      securityMgr)
   drivers(driverId) = driver
   driver.start()
   coresUsed += driverDesc.cores
   memoryUsed += driverDesc.mem
```

进入`driver.start()`方法，应用会创建`Driver`所需要的工作目录，同时`download`用户自定义的`jar`包 然后开始运行`Driver`。

```scala
/** Starts a thread to run and manage the driver. */
private[worker] def start() = {
  new Thread("DriverRunner for " + driverId) {
    override def run() {
        // prepare driver jars and run driver, 下载用户自定义的jar包, buildProcessBuilder该方法有两个默认值的备用参数，主要是准备程序运行的环境 (但并不包含app所在的jar)
        val exitCode = prepareAndRunDriver()
        // set final state depending on if forcibly killed and process exit code
        finalState = if (exitCode == 0) {
          Some(DriverState.FINISHED)
        } else if (killed) {
          Some(DriverState.KILLED)
        } else {
          Some(DriverState.FAILED)
        }
      // notify worker of final driver state, possible exception
      worker.send(DriverStateChanged(driverId, finalState.get, finalException))
    }
  }.start()
}
```

进一步进入到`prepareAndRunDriver()`方法，程序使用`CommandUtils.buildProcessBuilder()`结合`command`所要运行的环境，重新构建一个命令。例如: 本地环境变量、系统`classpath`, 替换掉传递过来的占位符。

```scala
private[worker] def prepareAndRunDriver(): Int = {
  val driverDir = createWorkingDirectory()
  val localJarFilename = downloadUserJar(driverDir)  // 下载用户自定义的jar包
  def substituteVariables(argument: String): String = argument match {
    case "{{WORKER_URL}}" => workerUrl
    case "{{USER_JAR}}" => localJarFilename
    case other => other
  }
  // TODO: If we add ability to submit multiple jars they should also be added here
  /* buildProcessBuilder该方法有两个默认值的备用参数，主要是准备程序运行的环境 (但并不包含app所在的jar) */
  val builder = CommandUtils.buildProcessBuilder(driverDesc.command, securityManager,
                                                 driverDesc.mem, sparkHome.getAbsolutePath, substituteVariables)
  runDriver(builder, driverDir, driverDesc.supervise)
}
```

进入`CommandUtils#buildLocalCommand`方法，`-cp`参数是在`buildCommandSeq(Command, Int, String)`中构建。

```scala
  /**
   * Build a command based on the given one, taking into account the local environment
   * of where this command is expected to run, substitute any placeholders, and append
   * any extra class paths.
   */
  private def buildLocalCommand(
      command: Command,
      securityMgr: SecurityManager,
      substituteArguments: String => String,
      classPath: Seq[String] = Seq.empty,
      env: Map[String, String]): Command = {
    val libraryPathName = Utils.libraryPathEnvName   // 返回系统的path，也就是一些
    val libraryPathEntries = command.libraryPathEntries
    val cmdLibraryPath = command.environment.get(libraryPathName)

    var newEnvironment = if (libraryPathEntries.nonEmpty && libraryPathName.nonEmpty) {
      val libraryPaths = libraryPathEntries ++ cmdLibraryPath ++ env.get(libraryPathName)
      command.environment + ((libraryPathName, libraryPaths.mkString(File.pathSeparator)))
    } else {
      /*
       * RestSubmissionClient发送过来的环境变量只有 SPARK_和MESOS_ 开头的环境变量，也即是对于driver端System.getenv()系统环境变量获取
       * 的值. 如spark-env初始化的 SPARK_ 开头的环境变量，在提交的时候已经创建好了;
       */
      command.environment
    }
    Command(
      /*
       * 对于driver并不是用户命令的入口，而是一个封装类org.apache.spark.deploy.DriverWrapper, 在封装类里面进一步解析
       *  对于executor是这个org.apache.spark.executor.CoarseGrainedExecutorBackend类
       */
      command.mainClass,
      command.arguments.map(substituteArguments),
      newEnvironment,
      command.classPathEntries ++ classPath,
      Seq.empty, // library path already captured in environment variable
      // filter out auth secret from java options
      command.javaOpts.filterNot(_.startsWith("-D" + SecurityManager.SPARK_AUTH_SECRET_CONF)))  // spark.jars在此处
  }
```

在`StandaloneRestServer#buildDriverDescription()`方法里指明如何构建`Command`类型，用命令行执行的是`org.apache.spark.deploy.worker.DriverWrapper`包装类。

```scala
/* 直接执行的是这个封装类，通过自定义urlClassLoader指定classpath的方式加载用户的jar然后通过反射执行 */
val command = new Command(
   "org.apache.spark.deploy.worker.DriverWrapper",
   Seq("{{WORKER_URL}}", "{{USER_JAR}}", mainClass) ++ appArgs, // args to the DriverWrapper
   environmentVariables, extraClassPath, extraLibraryPath, javaOpts)  // 也即是此时spark.jars也即--jars传来的参数在javaOpts里面
```

进入到`DriverManager#main(args: Array[String])`方法，通过自定义的`classLoader`加载`jar`包，根据`mainClass`通过反射执行其`main()`方法，触发用户程序的执行。

```scala
def main(args: Array[String]) {
  case workerUrl :: userJar :: mainClass :: extraArgs =>
  	    val conf = new SparkConf()
        val host: String = Utils.localHostName()
        val port: Int = sys.props.getOrElse("spark.driver.port", "0").toInt
        val rpcEnv = RpcEnv.create("Driver", host, port, conf, new SecurityManager(conf))
        logInfo(s"Driver address: ${rpcEnv.address}")
        rpcEnv.setupEndpoint("workerWatcher", new WorkerWatcher(rpcEnv, workerUrl))

        val currentLoader = Thread.currentThread.getContextClassLoader
        val userJarUrl = new File(userJar).toURI().toURL()
        val loader =
          if (sys.props.getOrElse("spark.driver.userClassPathFirst", "false").toBoolean) {
            new ChildFirstURLClassLoader(Array(userJarUrl), currentLoader)
          } else {
            new MutableURLClassLoader(Array(userJarUrl), currentLoader)
          }
        /*
         * 此时通过反射从userJarURL获取用户入口代码，调用用户的入口程序，然后执行. 在初始化SparkContext的时候会把spark.jars
         * 所指定的所有jar都添加到集群中 为将来执行tasks准备好依赖环境, return c.newInstance()
         */
        Thread.currentThread.setContextClassLoader(loader)
        setupDependencies(loader, userJar)

        // Delegate to supplied main class
        val clazz = Utils.classForName(mainClass)
        val mainMethod = clazz.getMethod("main", classOf[Array[String]])
        mainMethod.invoke(null, extraArgs.toArray[String])
        rpcEnv.shutdown()
}
```

现在`Driver`已经启动了，接下来看应用如何启动`executor`和`task`的流程，`Executor`的启动从`SparkContext#createTaskScheduler(SparkContext, String, String)`方法，方法体中会初始化`StandaloneSchedulerBackend`类。`SparkContext`准备完成后会调用`_taskScheduler.start()`方法启动`StandaloneSchedulerBackend`方法：

```scala
// start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler's
// constructor (YarnSchedule)
_taskScheduler.start()

private def createTaskScheduler(
  sc: SparkContext,
  master: String,
  deployMode: String): (SchedulerBackend, TaskScheduler) = {
    case SPARK_REGEX(sparkUrl) =>
    val scheduler = new TaskSchedulerImpl(sc) /* standalone模式下执行任务调度器executor */
    val masterUrls = sparkUrl.split(",").map("spark://" + _)
    /* 重点, 用户程序向master注册, executor申请都是由该函数完成的. start是在TaskSchedulerImpl中的start函数里启动的 */
    val backend = new StandaloneSchedulerBackend(scheduler, sc, masterUrls)
    scheduler.initialize(backend)
    (backend, scheduler)
}
```

进入`StandaloneSchedulerBackend#start()`方法，用`CoarseGrainedExecutorBackend`构建`command`命令，然后构建`ApplicationDescription`对象，将其传入`appClient`并向`Master`发起应用注册的请求`StandaloneAppClient#tryRegisterAllMasters()`方法中发送`RegisterApplication(appDescription, self)`，`Master`端收到请求后会重新运行`schedule()`的方法。

```scala
override def start() {
  super.start() 
    // Start executors with a few necessary configs for registering with the scheduler
    /* 只获取了Executor启动时用到的配置，不包含--jars传递的值 */
    val sparkJavaOpts = Utils.sparkJavaOpts(conf, SparkConf.isExecutorStartupConf)
    val javaOpts = sparkJavaOpts ++ extraJavaOpts
    val command = Command("org.apache.spark.executor.CoarseGrainedExecutorBackend",
      args, sc.executorEnvs, classPathEntries ++ testingClassPath, libraryPathEntries, javaOpts)
    // 重点关注两个参数 spark.executor.extraLibraryPath spark.driver.extraLibraryPath
    val webUrl = sc.ui.map(_.webUrl).getOrElse("")
    val coresPerExecutor = conf.getOption("spark.executor.cores").map(_.toInt)
	
  	val appDesc = ApplicationDescription(sc.appName, maxCores, sc.executorMemory, command,
      webUrl, sc.eventLogDir, sc.eventLogCodec, coresPerExecutor, initialExecutorLimit)
    client = new StandaloneAppClient(sc.env.rpcEnv, masters, appDesc, this, conf)
    client.start()
}
```

进入`Worker#receive()`方法，根据`case`匹配到`LaunchExecutor`的请求，构建`ExecutorRunner`对象并调用其`start()`方法。

```scala
case LaunchExecutor(masterUrl, appId, execId, appDesc, cores_, memory_) =>
	if (masterUrl != activeMasterUrl) {
     logWarning("Invalid Master (" + masterUrl + ") attempted to launch executor.")
  } else {
    logInfo("Asked to launch executor %s/%d for %s".format(appId, execId, appDesc.name))
    val manager = new ExecutorRunner(
      appId,
      execId,
      appDesc.copy(command = Worker.maybeUpdateSSLSettings(appDesc.command, conf)),
      cores_,
      memory_,
      self,
      workerId,
      host,
      webUi.boundPort,
      publicAddress,
      sparkHome,
      executorDir,
      workerUri,
      conf,
      appLocalDirs, ExecutorState.RUNNING)
    executors(appId + "/" + execId) = manager
    manager.start()
    coresUsed += cores_
    memoryUsed += memory_
    sendToMaster(ExecutorStateChanged(appId, execId, manager.state, None, None))
  }
```

进入`ExecutorRunner#start()`方法，首先创建了一个`worker`线程用于执行任务，要执行的方法为`fetchAndRunExecutor()`。在方法中通过`CommandUtils.buildProcessBuilder()`创建进程，然后设置执行路径、环境变量以及`spark UI`相关内容，然后启动进程（`process`执行类为`CoarseGrainedExecutorBackend`）。

```scala
/**
 * Download and run the executor described in our ApplicationDescription
 */
private def fetchAndRunExecutor() {
	// Launch the process
  val subsOpts = appDesc.command.javaOpts.map {
    Utils.substituteAppNExecIds(_, appId, execId.toString)
  }
  val subsCommand = appDesc.command.copy(javaOpts = subsOpts)
  val builder = CommandUtils.buildProcessBuilder(subsCommand, new SecurityManager(conf),
                                                 memory, sparkHome.getAbsolutePath, substituteVariables)
  val command = builder.command()
  val formattedCommand = command.asScala.mkString("\"", "\" \"", "\"")
  logInfo(s"Launch command: $formattedCommand")
  // 执行构建完成的ProcessBuilder
  process = builder.start()
  val header = "Spark Executor Command: %s\n%s\n\n".format(
  formattedCommand, "=" * 40)
  // Wait for it to exit; executor may exit with code 0 (when driver instructs it to shutdown)
  // or with nonzero exit code
  val exitCode = process.waitFor()
  state = ExecutorState.EXITED
  val message = "Command exited with code " + exitCode
  worker.send(ExecutorStateChanged(appId, execId, state, Some(message), Some(exitCode)))
}
```

在`CoarseGrainedExecutorBackend#receive()`方法中接收`case LaunchTask(data)`的请求，当`executor`初始化好之后执行`executor.launchTask(this, taskDesc)`方法。

```scala
override def receive: PartialFunction[Any, Unit] = {
  case LaunchTask(data) =>
  if (executor == null) {
    exitExecutor(1, "Received LaunchTask command but executor was null")
  } else {
    val taskDesc = TaskDescription.decode(data.value)
    logInfo("Got assigned task " + taskDesc.taskId)
    executor.launchTask(this, taskDesc)
  }
}
```

进入`TaskRunner#run()`方法，设置`TaskMemoryManager`、序列化`jar`文件、初始化各种`Metrics`统计信息，然后通过`task.run()`的任务就正常执行了。至此，从使用`spark-submit.sh`脚本提交用户`application`在`standalone`模式下的流程就先分析完成。

```scala
override def run(): Unit = {
  val taskMemoryManager = new TaskMemoryManager(env.memoryManager, taskId)
  val deserializeStartTime = System.currentTimeMillis()
  val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime
  } else 0L
  /*
   * 类加载器设置的是url类加载器, 而其父类加载器是系统类加载器. currentJars是以来的uri, 用户在调用
   * updateDependencies将依赖添加至此
   */
  Thread.currentThread.setContextClassLoader(replClassLoader)
  val ser = env.closureSerializer.newInstance()
  logInfo(s"Running $taskName (TID $taskId)")
  // Run the actual task and measure its runtime.
  taskStartTime = System.currentTimeMillis()
  taskStartCpu = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime
  } else 0L
  var threwException = true
  val value = Utils.tryWithSafeFinally {
    val res = task.run(
      taskAttemptId = taskId,
      attemptNumber = taskDescription.attemptNumber,
      metricsSystem = env.metricsSystem)
    threwException = false
    res
  } 
}
```

