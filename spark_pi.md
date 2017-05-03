

### Reference

- [Inintial Setup, a bit thin on words](https://dqydj.com/raspberry-pi-hadoop-cluster-apache-spark-yarn/)
- [Spark on Pi Blog](https://darrenjw2.wordpress.com/2015/04/17/installing-apache-spark-on-a-raspberry-pi-2/)
- [Spinning up Spark Cluster](http://blog.insightdatalabs.com/spark-cluster-step-by-step/#install-spark)
- Spark Configuration
    - [Standalone vs Client vs Cluster](http://www.agildata.com/apache-spark-cluster-managers-yarn-mesos-or-standalone/)
    - [Official Docs: Spark on YARN](http://spark.apache.org/docs/latest/running-on-yarn.html)



# Setting up Spark on the Pi Hadoop Cluster
Spark must be setup on all nodes, also since Spark is going to be run with hduser we will need to give hduser root access (or Spark will not be able to run properly). Also, we should add swap to each of the nodes to handle any memory overflow.

### Giving hduser root privledges
 To get spark to run properly the user must have sudo privledges. (This is important because if hduser is running hadoop and pi is running spark the two wont talk to each other i.e. spark wouldn't be using the hdfs. **I think**). We can grant sudo permission to hduser with the following as [explained here](https://askubuntu.com/questions/168280/how-do-i-grant-sudo-privileges-to-an-existing-user)

    sudo usermod -a -G sudo hduser

### Changing the Group and Owner of Spark to hduser
**After some review, I don't think granting hduser root privledge is strictly necessary...** convieniant yes, because I get tired of having to switch users to enter root permissions outside of hadoop, but checking the file permissions it seems that hduser never had group or owner permissions for spark. Hence the errors. Well this is a learning experience so n00b Linux mistakes are part of it. For a visual see below

    # what we have
    hduser@node1:/opt $ ls -la
    drwxr-xr-x 11 hduser hadoop 4096 Jan 11 07:54 hadoop
    drwxr-xr-x 16 pi     pi     4096 Apr 23 00:18 spark

    # what we want
    hduser@node1:/opt $ ls -la
    drwxr-xr-x 11 hduser hadoop 4096 Jan 11 07:54 hadoop
    drwxr-xr-x 16 hduser hadoop 4096 Apr 23 00:18 spark


Recalling that we created the group hadoop with user hduser as
    sudo addgroup hadoop
    sudo adduser --ingroup hadoop hduser
    sudo adduser hduser sudo

Then gave hduser ownership of the hadoop file using `sudo chown -R hduser.hadoop /opt/hadoop/`

We will repeat the process by giving hduser ownership of the spark file `/opt/spark` using the following. It is important to do this on all nodes!

    sudo chown -R hduser.hadoop /opt/spark


### Setting up Swap
See **swap.md** for how to setup swap on the pi

### Download and Install Spark
Download the latest version of Spark for Hadoop 2.7 (since the pi cluster uses Hadoop is 2.7.3) as a tgz file. The Spark downloads are located [here](http://spark.apache.org/downloads.html). I chose Spark 2.1 for Hadoop 2.7 i.e. `spark-2.1.0-bin-hadoop2.7.tgz`

The file can be un-gzipped using `sudo tar -xzf spark-2.1.0-bin-hadoop2.7.tgz` to create the directory with the same name. We will move this to our `/opt/` directory and rename to spark. Now our hadoop and spark installs are both in the `/opt/` directory.

    sudo mv spark-2.1.0-bin-hadoop2.7 /opt/spark

### Spinning up the Cluster
Provided all the permissions are set and the nodes can ssh to each other (recall that we set this up for hadoop using ssh over our eth0 network). We should be able to run the scripts in `/opt/spark/sbin` to start the Spark master and worker nodes. [Reference](http://spark.apache.org/docs/latest/spark-standalone.html#cluster-launch-scripts)

The easiest way to do this is to just run the `start-all.sh` script. This requires that we create a "slaves" file in `/opt/spark/conf/slaves` as described below
> To launch a Spark standalone cluster with the launch scripts, you should create a file called conf/slaves in your Spark directory, which must contain the hostnames of all the machines where you intend to start Spark workers, one per line. If conf/slaves does not exist, the launch scripts defaults to a single machine (localhost), which is useful for testing. Note, the master machine accesses each of the worker machines via ssh. By default, ssh is run in parallel and requires password-less (using a private key) access to be setup.

This is performed as follows

    # create the slaves file
    sudo nano /opt/spark/conf/slaves

    # enter the following into the slaves file and save (note this could easily have been the addresses instead of the host name, if both were entered it would start 2 workers on each node 1 for the host name one for the address)
    node1
    node2
    node3

We can then start the cluster using

    /opt/spark/sbin/start-all.sh

We can then monitor the Spark cluster via `192.168.1.165:8080`. If we want to shut down the cluster we would use the `stop-all.sh` script

    /opt/spark/sbin/stop-all.sh


## Spark Config
Also I completely skipped any kind of Spark config setup. This is almost certainly a mistake. [Quora has a great post on a Spark on cluster config](https://www.quora.com/How-do-I-set-up-Apache-Spark-with-Yarn-Cluster).

## Testing Spark Shell

We still have some errors, but much much better than before

    pi@node1:/opt/spark $ bin/spark-shell --master local[3]
    Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
    Setting default log level to "WARN".
    To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
    17/01/18 13:37:39 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    17/01/18 13:38:11 WARN ObjectStore: Version information not found in metastore. hive.metastore.schema.verification is not enabled so recording the schema version 1.2.0
    17/01/18 13:38:11 WARN ObjectStore: Failed to get database default, returning NoSuchObjectException
    17/01/18 13:38:16 WARN ObjectStore: Failed to get database global_temp, returning NoSuchObjectException
    Spark context Web UI available at http://192.168.0.110:4040
    Spark context available as 'sc' (master = local[3], app id = local-1484764662682).
    Spark session available as 'spark'.
    Welcome to
          ____              __
         / __/__  ___ _____/ /__
        _\ \/ _ \/ _ `/ __/  '_/
       /___/ .__/\_,_/_/ /_/\_\   version 2.1.0
          /_/

    Using Scala version 2.11.8 (Java HotSpot(TM) Client VM, Java 1.8.0_65)
    Type in expressions to have them evaluated.
    Type :help for more information.

    scala>__

It appears the test case for the spark shell works:

    scala> sc.textFile("README.md").count
    17/01/18 13:43:15 WARN SizeEstimator: Failed to check whether UseCompressedOops is set; assuming yes
    res0: Long = 104   

We can exit the program with `Ctrl+D`

## Checking PySpark
We can now check if the program works as expected with the PySpark shell using. Once again we see that having rw file permissions was the issue. If the program starts as expected we will see

    pi@node1:/opt/spark $ bin/pyspark --master local[3]
    Python 2.7.9 (default, Sep 17 2016, 20:26:04)
    [GCC 4.9.2] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
    Setting default log level to "WARN".
    To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
    17/01/18 13:48:08 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    17/01/18 13:48:35 WARN ObjectStore: Failed to get database global_temp, returning NoSuchObjectException
    Welcome to
          ____              __
         / __/__  ___ _____/ /__
        _\ \/ _ \/ _ `/ __/  '_/
       /__ / .__/\_,_/_/ /_/\_\   version 2.1.0
          /_/

    Using Python version 2.7.9 (default, Sep 17 2016 20:26:04)
    SparkSession available as 'spark'.
    >>>__

We then test the program using the following

    >>> sc.textFile("README.md").count()
    17/01/18 13:49:59 WARN SizeEstimator: Failed to check whether UseCompressedOops is set; assuming yes
    104     

Once again it appears to be successful.

# How can I verify this is working on all nodes!
- spark has no config updates from the download
- spark is only on the master node of the cluster
- the browser based port is using eth0 not wlan0 (i.e. I cant check the progress currently)


### Viewing Spark via the Brower
 - spark performance can be viewed in the browser via `http://192.168.1.165:4040`, note that this is the node1 address with port `4040`. See documentation [here](http://spark.apache.org/docs/latest/monitoring.html)


### Spark Environment Variables - A look back at Hadoop Config
Directly from the official Spark Documentation
> Ensure that HADOOP_CONF_DIR or YARN_CONF_DIR points to the directory which contains the (client side) configuration files for the Hadoop cluster.

Neither of these variables are set as part of the Hadoop Setup, **what do we use instead?**. From Widrikson's tutorial (referenced in the Hadoop setup) we have the following enviornment variables:




It turns out this is actually in the system, which isnt wholly surprising given that it is the only env variable recommended in the HADOOP setup documentation! The variable is housed in `/opt/hadoop/etc/hadoop/hadoop-env.sh` and found on line 34 as `export HADOOP_CONF_DIR=${HADOOP_CONF_DIR:-"/etc/hadoop"}`. However this file is not called!  running `source /opt/hadoop/etc/hadoop/hadoop-env.sh` changes HADOOP_CONF_DIR to `/etc/hadoop` while we have installed hadoop to `/opt/hadoop`. Trying to start or stop a cluster no longer works as expected!

It looks a bit like this is all a bit of a miscommunication in the setup, both hadoop and yarn are setup via the `/etc/bash.bashrc` to use `/opt/hadoop` but that file does not have the two recommended envirnment variables HADOOP_CONF_DIR and YARN_CONF_DIR. So we will simply go into `/etc/bash.bashrc` and just add these variables to use `/opt/hadoop` as the system would expect anyway!

    I also have a two line ‘spark-env.sh’ file, which is inside the ‘conf’ directory of Spark:

    SPARK_MASTER_IP=192.168.1.50
    SPARK_WORKER_MEMORY=512m

    (I’m not even sure this is necessary, since we will be running on YARN)


### Enviornment Variables & Errors
 **NativeCodeLoader Error**

    WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable

This is caused when the path for the hadoop native library files is not found, it is resolved by setting the following Environment variable:



### Spark Environment Variables
There is some mixed guidance on this topic including the quote below from (DQYDJ)[https://dqydj.com/raspberry-pi-hadoop-cluster-apache-spark-yarn/]

>I also have a two line ‘spark-env.sh’ file, which is inside the ‘conf’ directory of Spark: (I’m not even sure this is necessary, since we will be running on YARN)

>SPARK_MASTER_IP=192.168.1.50
>SPARK_WORKER_MEMORY=512m

Then directly from the Apache [Running Spark on YARN Documentation](http://spark.apache.org/docs/latest/running-on-yarn.html)

> Ensure that HADOOP_CONF_DIR or YARN_CONF_DIR points to the directory which contains the (client side) configuration files for the Hadoop cluster.

We also want to see the Spark configuration

### Spark Configuration
spark configuration is `/opt/spark/conf/spark-env.sh.template`

After some reading it seems that we want to run Spark in `standalone mode` as it will still take advantage of the cluster (as opposed to Spark on YARN). The official standalone setup documents are [here](http://spark.apache.org/docs/latest/spark-standalone.html)

The official guidance for Standalone Mode is:
> To install Spark Standalone mode, you simply place a compiled version of Spark on each node on the cluster.

Without configuring any of the Worker nodes the default values are set to 1 GB of memory each (1024 MB) and 4 Cores.



##  Starting the Spark Cluster
The immediate things that need to be addressed such that we can use simple scripts such as `/opt/spark/start-all.sh` or `/opt/spark/start-slaves.sh` is to fix the Permission denied issue when this is run.





## Opening a Spark Session
Now that the culster is spun up it is awaiting commands from an application. To open a console to interact with Spark directly we can run:

    /opt/spark/bin/spark-shell --master spark://node1:7077

We can check the progress of this session using the browser at `http://node1:8080` which will confirm the (internal address?) and the master and slave nodes, along with their address, number of cores, and memory. (note this is standalone mode)






>The statement below is unecessary!
>There are two deploy modes that can be used to launch Spark on YARN, `cluster` and `client` mode (which are different than Spark `Mesos` and `standalone` mode).
