---
layout: post
title: Getting Spark on Windows to connect to AWS EMR cluster
---
I managed to get Spark to run on Windows in local mode, and to submit jobs to an EMR cluster in AWS.

Here are all the issues I had to work through.

*Getting Spark to run on Windows in general*

* First, download the Spark and Hadoop binaries.  
* Make sure you have appropriate JDK installed and JAVA_HOME environment set properly.  Windows will struggle with paths that contain spaces, so best to install or link it from somewhere else.
* Solving the winutils.exe dependency ("Could not locate executable null\bin\winutils.exe")
    * download [this exe](https://github.com/steveloughran/winutils/raw/master/hadoop-2.6.0/bin/winutils.exe) and install it somewhere in a 'bin' subfolder
    * set `HADOOP_HOME` environment var to where you put that exe - Spark will look for `%HADOOP_HOME%\bin\winutils.exe` so make sure you don't include 'bin' in the `HADOOP_HOME` var!
* Solving permissions on \tmp\hive ("The root scratch dir: /tmp/hive on HDFS should be writable. Current permissions are: ...")
    * solution: `%HADOOP_HOME%\bin\winutils.exe chmod 777 \tmp\hive`
    * Note that it is important to use the 64-bit version of winutils if you are on a 64-bit system.  Otherwise it will look like permissions are fixed, but they aren't
        * See my comment [on this post](https://stackoverflow.com/questions/34196302/the-root-scratch-dir-tmp-hive-on-hdfs-should-be-writable-current-permissions)

*Connecting to EMR* 

* Set up bare-bones Hadoop config files - only need settings to specify how client connects to cluster.  (Assumes the EMR cluster's security group allows your workstation to connect in the first place.)

```
<!-- yarn-site.xml-->
 
<configuration>
<!-- Site specific YARN configuration properties -->
<!-- your hostnames/port numbers will be different -->
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>your-emr-cluster:8032</value>
  </property>
  <property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>your-emr-cluster:8030</value>
  </property>
  <property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>your-emr-cluster:8031</value>
  </property>
  <property>
    <name>yarn.resourcemanager.admin.address</name>
    <value>your-emr-cluster:8033</value>
  </property>
  <property>
    <name>yarn.resourcemanager.webapp.adress</name>
    <value>your-emr-cluster:8088</value>
  </property>
  <property>
    <name>yarn.web-proxy.address</name>
    <value>your-emr-cluster:20888</value>
  </property>
</configuration>
 

<!-- core-site.xml -->
<configuration>
<property>
        <name>fs.defaultFS</name>
        <value>hdfs://your-emr-cluster:8020</value>
    </property>
</configuration>
```

* set `HADOOP_CONF_DIR` to the path where those XML files live
* set `HADOOP_USER_NAME to` override OS user, to avoid "Permission denied: user=xxxx, access=WRITE, inode="/user/xxxx/.sparkStaging/application_xxxxx":hdfs:hdfs:drwxr-xr-x"  (Assumes 'simple'/no authentication on cluster)
* run `spark-submit` with `--master yarn --deploy-mode cluster`
* `--deploy-mode client` will NOT work unless your EMR cluster has a route back to your workstation.  Otherwise, you will see your jobs stuck in ACCEPTED stage in the YARN resource manager.  This means that spark-shell and spark-sql will not work.