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
      case m if m.startsWith("spark") => STANDALONE
      case m if m.startsWith("mesos") => MESOS
      case m if m.startsWith("k8s") => KUBERNETES
      case m if m.startsWith("local") => LOCAL
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
      /* YARN_CLUSTER_SUBMIT_CLASS为org.apache.spark.deploy.yarn.YarnClusterApplication */
      childMainClass = YARN_CLUSTER_SUBMIT_CLASS
    } 
    (childArgs, childClasspath, sparkConf, childMainClass) 
}
```

