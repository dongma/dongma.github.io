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

```scala
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

进入`YarnClientSchedulerBackend#start()`方法，创建`client`对象去提交任务，然后调用`client.submitApplication()`使用`AM`向`ResourceManager`申请资源。在`super.start()`中会启动`CoarseGrainedSchedulerBackend`，等待`app`的启动成功。

```scala
override def start() {
  /* 动态申请资源的时候才会调用 SchedulerBackendUtils#getInitialTargetExecutorNumber */
  totalExpectedExecutors = SchedulerBackendUtils.getInitialTargetExecutorNumber(conf)
  client = new Client(args, conf)
  /* 将Application提交之后  # 可看ApplicationMaster#main()的启动 */
  bindToYarn(client.submitApplication(), None)
  // SPARK-8687: Ensure all necessary properties have already been set before
  // we initialize our driver scheduler backend, which serves these properties
  // to the executors
  /* 调用YarnSchedulerBackend的父类CoarseGrainedSchedulerBackend#start()方法，在start()方法里实现自己 */
  super.start()
  waitForApplication()
}
```

进一步看`client.submitApplication()`提交应用给`AppMaster`前，如何初始化`ContainerContext`运行环境、`java opts`和运行`AM`的指令，进入`createContainerLaunchContext()`方法，`client`模式下`amClass`为`org.apache.spark.deploy.yarn.ExecutorLauncher`。在`yarn client`模式下，都是有`appMaster`向`resourceManager`申请`--num-executor NUM`参数指定的数目。

```scala
/**
 * Set up a ContainerLaunchContext to launch our ApplicationMaster container.
 * This sets up the launch environment, java options, and the command for launching the AM.
 */
private def createContainerLaunchContext(newAppResponse: GetNewApplicationResponse) {
  // 设置环境变量及spark-java-opts
  val launchEnv = setupLaunchEnv(appStagingDirPath, pySparkArchives)
  /*
   * 这个函数的主要作用是将用户自己打的jar包(--jars指定的jar发送到分布式缓存中去)，并设置了spark.yarn.user.jar
   * 和spark.yarn.secondary.jars这两个参数, 然后这两个参数会被封装程 --user-class-path 传递给
   * executor使用
   */
  val localResources = prepareLocalResources(appStagingDirPath, pySparkArchives)
  val amContainer = Records.newRecord(classOf[ContainerLaunchContext])
  amContainer.setLocalResources(localResources.asJava)
  amContainer.setEnvironment(launchEnv.asJava)
  // Add Xmx for AM memory
  javaOpts += "-Xmx" + amMemory + "m"
  val tmpDir = new Path(Environment.PWD.$$(), YarnConfiguration.DEFAULT_CONTAINER_TEMP_DIR)
  javaOpts += "-Djava.io.tmpdir=" + tmpDir
  /* 判断是否在cluster集群环境来确定AMclass, client模式下为ExecutorLauncher, 通过AMclass及一些参数构建command 进而构建amContainer */
  val amClass =
  if (isClusterMode) {
    Utils.classForName("org.apache.spark.deploy.yarn.ApplicationMaster").getName
  } else {
    Utils.classForName("org.apache.spark.deploy.yarn.ExecutorLauncher").getName
  }
  amContainer
}
```

在`super.start()`需要重点看一下`YarnSchedulerBackend`的父类`CoarseGrainedSchedulerBackend`的`start()`方法，方法体内创建了一个`driverEndpoint`的`RPC`客户端。在`YarnSchedulerBackend`类中覆盖了`createDriverEndpointRef()`方法，用子类`YarnDriverEndpoint`替代`DriverEndpoint`并重写了其`onDisconnected()`方法（是由于协议的不同）。

```scala
/* YarnSchedulerBackend启动时实例化，负责根ApplicationMaster进行通信 */
private val yarnSchedulerEndpoint = new YarnSchedulerEndpoint(rpcEnv)
override def start() {
	// TODO (prashant) send conf instead of properties
	driverEndpoint = createDriverEndpointRef(properties)
}
```

`yarn-client`代码分析完之后，进入`ApplicationMaster#main(Array[String])`，在上文`client#createContainerLaunchContext()`时，指定`amClass`为`org.apache.spark.deploy.yarn.ExecutorLauncher`（`main`方法中封装了`ApplicationMaster`），最终调用`runExecutorLauncher()`运行`executor`。

```scala
private def runExecutorLauncher(): Unit = {
  val hostname = Utils.localHostName
  val amCores = sparkConf.get(AM_CORES)
  rpcEnv = RpcEnv.create("sparkYarnAM", hostname, hostname, -1, sparkConf, securityMgr,
                         amCores, true)
  // The client-mode AM doesn't listen for incoming connections, so report an invalid port.
  registerAM(hostname, -1, sparkConf, sparkConf.getOption("spark.driver.appUIAddress"))
  // The driver should be up and listening, so unlike cluster mode, just try to connect to it
  // with no waiting or retrying.
  val (driverHost, driverPort) = Utils.parseHostPort(args.userArgs(0))
  val driverRef = rpcEnv.setupEndpointRef(
    RpcAddress(driverHost, driverPort),
    YarnSchedulerBackend.ENDPOINT_NAME)
  addAmIpFilter(Some(driverRef))
  /* 向resourceManager申请根启动--num-executor相同的资源 */
  createAllocator(driverRef, sparkConf)
  // In client mode the actor will stop the reporter thread.
  reporterThread.join()
}
```

在`appMaster#createAllocator()`会进入到`allocator#allocateResources()`申请资源，接着进入`handleAllocatedContainers(Seq[Container])`方法。在`runAllocatedContainers()`中在已经申请到的`container`中运行`executor`。

```scala
/**
 * Launches executors in the allocated containers.
 */
private def runAllocatedContainers(containersToUse: ArrayBuffer[Container]): Unit = {
  new ExecutorRunnable(
    Some(container), conf, sparkConf, driverUrl, executorId, executorHostname,executorMemory,
    executorCores, appAttemptId.getApplicationId.toString, securityMgr, localResources
  ).run()
}
```

在`ExecutorRunnable#startContainer()`中会设置本地相关环境变量，然后`nmClient`会启动`container`。

```scala
def startContainer(): java.util.Map[String, ByteBuffer] = {
  /* 此处设置spark.executor.extraClassPath为系统环境变量 */
  ctx.setLocalResources(localResources.asJava)
  // Send the start request to the ContainerManager
  try {
    nmClient.startContainer(container.get, ctx)
  } catch {
    case ex: Exception =>
    throw new SparkException(s"Exception while starting container ${container.get.getId}" +
                             s" on host $hostname", ex)
  }
}
```

在`CoarseGrainedExecutorBackend#main(Array[String])`启动时会执行`run(driverUrl, executorId, hostname, cores, appId, workerUrl, userClassPath)`的方法。先创建`env`然后根据`env`使用`CoarseGrainedExecutorBackend`作为`executor`创建`rpc`。

```scala
/* 创建env主要用与Rpc提交相关的请求 */
val env = SparkEnv.createExecutorEnv(
driverConf, executorId, hostname, cores, cfg.ioEncryptionKey, isLocal = false)

env.rpcEnv.setupEndpoint("Executor", new CoarseGrainedExecutorBackend(
	env.rpcEnv, driverUrl, executorId, hostname, cores, userClassPath, env))
workerUrl.foreach { url =>
	env.rpcEnv.setupEndpoint("WorkerWatcher", new WorkerWatcher(env.rpcEnv, url))
}
env.rpcEnv.awaitTermination()
```

`rpc`在`onStart()`的时候会发送`RegisterExecutor`的请求，用于注册`executor`的相关信息。

```scala
override def onStart() {
  logInfo("Connecting to driver: " + driverUrl)
  rpcEnv.asyncSetupEndpointRefByURI(driverUrl).flatMap { ref =>
    // This is a very fast action so we can use "ThreadUtils.sameThread"
    driver = Some(ref)
    ref.ask[Boolean](RegisterExecutor(executorId, self, hostname, cores, extractLogUrls))
  }(ThreadUtils.sameThread).onComplete {
    // This is a very fast action so we can use "ThreadUtils.sameThread"
    case Success(msg) =>
    // Always receive `true`. Just ignore it
    case Failure(e) =>
    exitExecutor(1, s"Cannot register with driver: $driverUrl", e, notifyDriver = false)
  }(ThreadUtils.sameThread)
}
```

`Driver`端`CoarseGrainedSchedulerBackend#receiveAndReply(RpcCallContext)`在收到`executor`注册请求时，会`reply`一个已经注册成功的响应。

```scala
executorRef.send(RegisteredExecutor)
// Note: some tests expect the reply to come after we put the executor in the map
context.reply(true)
listenerBus.post(
SparkListenerExecutorAdded(System.currentTimeMillis(), executorId, data))
```

`executor`收到响应后会启动一个`exectuor`，接下来就是等待`Driver`发送过来要进行调度的任务（用`case LaunchTask`匹配请求）。`executor`执行`launchTask()`，创建`TaskRunner`任务运行的流程就与`standalone`模式相同，`yarn-client`模式下`spark`任务提交以及运行的流程就是这样。

```scala
override def receive: PartialFunction[Any, Unit] = {
  /* Driver响应executor注册成功时接收的请求 */
  case RegisteredExecutor =>
    logInfo("Successfully registered with driver")
    try {
      executor = new Executor(executorId, hostname, env, userClassPath, isLocal = false)
    } catch {
      case NonFatal(e) =>
      exitExecutor(1, "Unable to create executor due to " + e.getMessage, e)
    } 
  /* Driver发送过来要进行调度的任务 */
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



