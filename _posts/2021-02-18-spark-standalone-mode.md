---
layout: post
title: spark standalone模式启动源码分析
---
## spark standalone模式启动源码分析

> spark目前支持以standalone、Mesos、YARN、Kubernetes等方式部署，本文主要分析apache spark在standalone模式下资源的初始化、用户application提交，在spark-submit脚本提交应用时，如何将--extraClassPath等参数传递给Driver等相关流程。

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



