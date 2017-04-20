# wlan0 appears not to work eth0 default port instead
I may need to go back and fix the 255.255.255.255 that was "fixed" in the config file to get my pi's back on the right network... i.e. on wlan0 such that I can use the browser to check their status!

# Spark Install on a 3 node Raspberry Pi Cluster

- [Inintial Setup, a bit thin on words](https://dqydj.com/raspberry-pi-hadoop-cluster-apache-spark-yarn/)
- [Spark on Pi Blog](https://darrenjw2.wordpress.com/2015/04/17/installing-apache-spark-on-a-raspberry-pi-2/)

## Test the Spark Shell


To test the spark shell run the following `bin/spark-shell --master local[4]` (DO WE WANT local[3] since we only have 3 nodes).

Note this generated the error `/opt/spark/metastore_db cannot be created` which eats a ton of screen real estate. This post looks like it may address the issue
[Spark Shell doesnt start correctly](http://stackoverflow.com/questions/36273166/spark-shell-doesnt-start-correctly-spark1-6-1-bin-hadoop2-6-version). **I haven't read it yet**. Eventually however the Spark Prompt did appear.

    Welcome to
          ____              __
         / __/__  ___ _____/ /__
        _\ \/ _ \/ _ `/ __/  '_/
       /___/ .__/\_,_/_/ /_/\_\   version 2.1.0
          /_/

    Using Scala version 2.11.8 (Java HotSpot(TM) Client VM, Java 1.8.0_65)
    Type in expressions to have them evaluated.
    Type :help for more information.__


Running the test command `sc.textFile("README.md").count` produced an error




    scala> sc.textFile("README.md").count
    <console>:18: error: not found: value sc
           sc.textFile("README.md").count
           ^


## Test the PySpark Console

after running the <> command we have the same error as before

    17/01/18 13:24:34 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    Wed Jan 18 13:24:43 EST 2017 Thread[Thread-2,5,main] java.io.FileNotFoundException: derby.log (Permission denied)
    Wed Jan 18 13:24:44 EST 2017 Thread[Thread-2,5,main] Cleanup action starting
    ERROR XBM0H: Directory /opt/spark/metastore_db cannot be created.

once again, it looks like we have the expected prompt, but spark throws an error on the test example. **The good news is if its the same error in both places perhaps we only have 1 thing to fix**

    >>> sc.textFile("README.md").count()
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    NameError: name 'sc' is not defined


# Updates

It appears that the issues above were simply due to file permissions, I had hoped this was the case but since nothing is ever really simple I doubted myself. Turns out things work out much better running from user `pi` rather than `hduser` which did not have root permissions and couldnt make the required db directory.

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


### Notes:
 in addition to the DQDYGFYS tutorial I found [this](http://spark.apache.org/docs/latest/running-on-yarn.html) on the official spark page. it loks like it connects the dots with the example from GAGFS
