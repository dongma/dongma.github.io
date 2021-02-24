---
layout: post
title: 分析spark在yarn-client和yarn-cluster模式下启动
---
## spark在yarn-client和yarn-cluster模式下启动

> 文章分析`spark`在`yarn-client`、`yarn-cluster`模式下启动的流程，`yarn`是`apache`开源的一个资源管理的组件。`JobTracker`在`yarn`中大致分为了三块：一部分是`ResourceManager`，负责`Scheduler`及`ApplicationsManager`；一部分是`ApplicationMaster`，负责`job`生命周期的管理；最后一部分是`JobHistoryServer`，负责日志的展示；

先看一个`spark`官网上通过`yarn`提交用户应用程序的`spark-submit`脚本，从该脚本开始分析在`yarn`环境下执行的流程。

```shell
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master yarn \
  --deploy-mode cluster \  # can be client for client mode
  --executor-memory 20G \
  --num-executors 50 \
  /path/to/examples.jar \
  1000
```

在分析源码前需要在父`pom.xml`中引入`yarn`资源代码模块，使得其`class`文件加载到`classpath`中。

```xml
<!-- See additional modules enabled by profiles below -->
<module>resource-managers/yarn</module>
```

与`standalone`模式应用启动一样，`SparkSubmit#runMain(SparkSubmitArguments, Boolean)`是应用程序的入口。由于是在`yarn`环境下启动，在前期准备`submit`环境时会有差异，差异点在`prepareSubmitEnvironment(SparkSubmitArguments, Option[HadoopConfiguration])`方法，在方法中会依`args.master`、`args.deployMode`进行模式匹配，当`master`为`yarn`时，会将`childMainClass`设置为`org.apache.spark.deploy.yarn.YarnClusterApplication`作为资源调度的启动类。

<!-- more -->

```scala
private[deploy] def prepareSubmitEnvironment(
   args: SparkSubmitArguments,
   conf: Option[HadoopConfiguration] = None)
   : (Seq[String], Seq[String], SparkConf, String) = {
    // Set the cluster manager
    val clusterManager: Int = args.master match {
      case "yarn" => YARN
      case "yarn-client" | "yarn-cluster" =>
        logWarning(s"Master ${args.master} is deprecated since 2.0." +
          " Please use master \"yarn\" with specified deploy mode instead.")
        YARN
    }	 
    if (deployMode == CLIENT) {
    /* 在client模式下 用户程序直接在submit内通过反射机制执行，此时用户自己打的jar和--jars指定的jar都会被加载到classpath中  */
    	childMainClass = args.mainClass
    	if (localPrimaryResource != null && isUserJar(localPrimaryResource)) {
       	childClasspath += localPrimaryResource
    	}
    	if (localJars != null) { childClasspath ++= localJars.split(",") }
    }
    // In yarn-cluster mode, use yarn.Client as a wrapper around the user class
    if (isYarnCluster) {
      /* YARN_CLUSTER_SUBMIT_CLASS在cluster模式下为org.apache.spark.deploy.yarn.YarnClusterApplication */
      childMainClass = YARN_CLUSTER_SUBMIT_CLASS
    } 
    (childArgs, childClasspath, sparkConf, childMainClass) 
}
```

`submit()`需要的环境准备好之后，通过`mainClass`构建`spark`应用，由于目前分析在`yarn client`模式下的启动，`mainClass`并不是`SparkApplication`的实例。因而，`app`类型为`JavaMainApplication`。

```
val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
      mainClass.newInstance().asInstanceOf[SparkApplication]
} else {
  // SPARK-4170
  if (classOf[scala.App].isAssignableFrom(mainClass)) {
  logWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
  }
  new JavaMainApplication(mainClass)
}
/* standalone模式在 SparkSubmit#prepareSubmitEnvironment(args)中将childMainClass设置为RestSubmissionClient */
app.start(childArgs.toArray, sparkConf)
```

在`start()`方法中会通过反射获取得到`main`方法，然后进行调用执行用户`jar`包中的代码。进入用户程序（`main`方法）之后，存在两个重要的类`SparkConf`和`SparkContext`，根据`config`配置信息实例化`context`上下文。

```scala
override def start(args: Array[String], conf: SparkConf): Unit = {
  val mainMethod = klass.getMethod("main", new Array[String](0).getClass)
  if (!Modifier.isStatic(mainMethod.getModifiers)) {
    throw new IllegalStateException("The main method in the given main class must be static")
  }
  val sysProps = conf.getAll.toMap
  sysProps.foreach { case (k, v) =>
    sys.props(k) = v
  }
  mainMethod.invoke(null, args)
}
/* spark application中使用sparkConf和sparkContext加载环境相关配置 */
val config = new SparkConf().setAppName("spark-app")
	.set("spark.app.id", "spark-mongo-connector")
val sparkContext = new SparkContext(config)
```

在`SparkContext#createTaskScheduler(SparkContext, String, String)`方法中会根据`master`确定`scheduler`和`backend`。由于`master`为`yarn`，在`getClusterManager(String)`中确定`cm`的类型为`YarnClusterManager`。在`yarn-client`模式下调用`createTaskScheduler()`和`createSchedulerBackend()`通过`masterUrl`和`deployMode`可得 `scheduler`为`YarnScheduler`、`backend`为`YarnClientSchedulerBackend`。

```scala
/* 当masterUrl为外部资源时 (Yarn、Mesos、K8s)，走此处的逻辑: (yarn)cluster模式走YarnClusterScheduler、
        (yarn)client走YarnScheduler用于资源调度 */
case masterUrl =>
  val cm = getClusterManager(masterUrl) match {
    case Some(clusterMgr) => clusterMgr
    case None => throw new SparkException("Could not parse Master URL: '" + master + "'")
  }
  val scheduler = cm.createTaskScheduler(sc, masterUrl)
  val backend = cm.createSchedulerBackend(sc, masterUrl, scheduler)
  cm.initialize(scheduler, backend)
```

进入`YarnClientSchedulerBackend#start()`方法，