# Hadoop Primer
This is a simple tutorial on starting, checking and running the canonical "Hello World" of Hadoop, namely the canned MapReduce WordCount program. Some basic tools for diagnosing the DFS and job progress are also presented below. The purpose of this is to simply have a reference for the basics, such that we can easily pick up our toy cluster and get up and running after time away.

## TL;DR
    # login and switch user
    ssh pi@192.168.1.165
    su hduser

    # start hadoop and check status
    start-dfs.sh
    start-yarn.sh
    jps

    # load a file to hdfs
    hadoop fs -put <filename> /<filename>

    # show hdfs directory contents
    hadoop fs -ls /

    # shutdown
    stop-dfs.sh
    stop-yarn.sh




### Log-in to our cluster
Recalling that the ssh handshake is setup between jonbruno@Jons-IMac and pi@node1, we can log into our headless cluster using `ssh pi@192.168.1.165`. The way the system is setup "pi" is the admin and "hduser" is the account with permissions for the Hadoop cluster. Therefore, we need to change the user to "hduser" using `su hduser` and enter the password at the prompt.

### Start Hadoop
The start and stop scripts are located in this directory `/opt/hadoop/sbin`. To start the distributed file system "dfs" we run `start-dfs.sh` and should see the following:

    hduser@node1:/home/pi $ start-dfs.sh

    Starting namenodes on [node1]
    node1: starting namenode, logging to /opt/hadoop/logs/hadoop-hduser-namenode-node1.out
    node1: starting datanode, logging to /opt/hadoop/logs/hadoop-hduser-datanode-node1.out
    node2: starting datanode, logging to /opt/hadoop/logs/hadoop-hduser-datanode-node2.out
    node3: starting datanode, logging to /opt/hadoop/logs/hadoop-hduser-datanode-node3.out
    Starting secondary namenodes [0.0.0.0]
    0.0.0.0: starting secondarynamenode, logging to /opt/hadoop/logs/hadoop-hduser-secondarynamenode-node1.out

*Note that these files appear to be on the hadoop path, as such we can run them from our home directory as per the example above.* To start the resource negotiator "yarn" we use `start-yarn.sh` and see the following:

    hduser@node1:/home/pi $ start-yarn.sh

    starting yarn daemons
    starting resourcemanager, logging to /opt/hadoop/logs/yarn-hduser-resourcemanager-node1.out
    node3: starting nodemanager, logging to /opt/hadoop/logs/yarn-hduser-nodemanager-node3.out
    node1: starting nodemanager, logging to /opt/hadoop/logs/yarn-hduser-nodemanager-node1.out
    node2: starting nodemanager, logging to /opt/hadoop/logs/yarn-hduser-nodemanager-node2.out

We can verify that everything is running correctly using `jps` which checks the NameNode, DataNode, ResourceManager, Jps, etc. are working. JPS stands for job processing scheduler. If everything is working correctly you should see output similar to the following:

    hduser@node1:/home/pi $ jps

    1153 NameNode
    2035 Jps
    1252 DataNode
    1609 ResourceManager
    1417 SecondaryNameNode
    1709 NodeManager

An alternative way to check that everything is running correctly is to use the following. It provides a more detailed description of the cluster.

    hdfs dfsadmin -report



### Load a file into HDFS
I'm going to use a sample file from a tutorial I used to get Hadoop setup on the Raspberry pi initially ([described here](http://www.widriksson.com/raspberry-pi-2-hadoop-2-cluster/#Run-hadoop-hello-world-wordcount)). First we download the files.

    cd ~
    wget http://www.widriksson.com/wp-content/uploads/2014/10/hadoop_sample_txtfiles.tar.gz
    tar -xvzf hadoop_sample_txtfiles.tar.gz ~/

Then we need to load them into HDFS

    hadoop fs -put smallfile.txt /smallfile.txt

We can then check the contents of HDFS, where we expect to see the two new directories

    hduser@node1:~ $ hadoop fs -put smallfile.txt /smallfile.txt
    hduser@node1:~ $ hadoop fs -ls /
    Found 1 items
    -rw-r--r--   3 hduser supergroup    1226438 2017-01-03 21:21 /smallfile.txt


Note that anything stored in the `tmp\` directory will be deleted upon a reboot. As such we need to save any data in a different directory should we want our data to persist for future sessions. See [this post](http://stackoverflow.com/questions/28379048/data-lost-after-shutting-down-hadoop-hdfs) for a better explaination.

### Running the Canonical Hadoop Wordcount Example
The path to the JAR file that runs the wordcount example, `hadoop-mapreduce-examples-2.7.3.jar `, can be a bit difficult to find. On my install the hadoop source file was downloaded to `/home/pi` thus the path to find the Wordcount JAR is

    /home/pi/hadoop-2.7.3-src/hadoop-mapreduce-project/hadoop-mapreduce-examples/target/hadoop-mapreduce-examples-2.7.3.jar

The command we want to use is as follows, note that `time` is optional and tells Hadoop to print the processing time to the command line

    time hadoop jar <path>/hadoop-mapreduce-examples-2.7.3.jar <program> <hdfs file in> <hdfs file out>

Executing the example `wordcount` program looks like:

    cd  /home/pi/hadoop-2.7.3-src/hadoop-mapreduce-project/hadoop-mapreduce-examples/target/
    time hadoop jar hadoop-mapreduce-examples-2.7.3.jar wordcount /smallfile.txt /small-file-out-1

Since we used `time` we can see how long it took to process our job

    real	1m29.284s
    user	0m13.180s
    sys	0m0.680s

We can also confirm that our job finished as expected by checking the hadoop filesystem

    hadoop fs -ls /small-file-out-1

    Found 3 items
    -rw-r--r--   3 hduser supergroup          0 2017-04-23 22:32 /small-file-out-1/_SUCCESS
    -rw-r--r--   3 hduser supergroup     106203 2017-04-23 22:32 /small-file-out-1/part-r-00000
    -rw-r--r--   3 hduser supergroup     106940 2017-04-23 22:32 /small-file-out-1/part-r-00001
    ._

We can also retrieve the file using the `hadoop fs -get <hdfs dir name> <local dir name>` command. For example if we want to move the file to our home directory

    hadoop fs -get /small-file-out-1 ~/small-file-out-1


This is a very simple example, the official MapReduce Tutorial can be found [here](https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html)


### Shutdown Hadoop and YARN
Shutting down is basically the same as starting up except that we now run shell scripts with stop in place of start. The scripts are located in `/opt/hadoop/sbin` Running `stop-hdfs.sh` produces:

    hduser@node1:/opt/hadoop/sbin $ stop-dfs.sh

    Stopping namenodes on [node1]
    node1: stopping namenode
    node1: stopping datanode
    node3: stopping datanode
    node2: stopping datanode
    Stopping secondary namenodes [0.0.0.0]
    0.0.0.0: stopping secondarynamenode

Then running `stop-yarn.sh` produces:

    hduser@node1:/opt/hadoop/sbin $ stop-yarn.sh

    stopping yarn daemons
    stopping resourcemanager
    node1: stopping nodemanager
    node3: no nodemanager to stop
    node2: no nodemanager to stop
    no proxyserver to stop
