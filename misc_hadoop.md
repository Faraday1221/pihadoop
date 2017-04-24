## Hadoop Environment Variables
I didn't do a particularly good job of documenting the hadoop environment variables. As a result there is now some confusion in my setup as I go to setup Spark. Below is a recap of the Hadoop Environment Variables used to setup hadoop

From a fresh reboot the following variables are loaded from `/etc/bash.bashrc`
    pi@node1:~ $ echo $JAVA_HOME
    /usr/lib/jvm/jdk-8-oracle-arm32-vfp-hflt/
    pi@node1:~ $ echo $HADOOP_INSTALL
    /opt/hadoop

The bash.bashrc file has the following lines

    export JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:jre/bin/java::")
    export HADOOP_INSTALL=/opt/hadoop
    export PATH=$PATH:$HADOOP_INSTALL/bin
    export PATH=$PATH:$HADOOP_INSTALL/sbin
    export HADOOP_MAPRED_HOME=$HADOOP_INSTALL
    export HADOOP_COMMON_HOME=$HADOOP_INSTALL
    export HADOOP_HDFS_HOME=$HADOOP_INSTALL
    export YARN_HOME=$HADOOP_INSTALL
    export HADOOP_HOME=$HADOOP_INSTALL

Something confusing is that the `/opt/hadoop/etc/hadoop/hadoop-env.sh` file has commands to set $JAVA_HOME and $HADOOP_CONF_DIR but it does not appear to be called during as part of either `start-dfs.sh` or `start-yarn.sh`. In fact `echo $HADOOP_CONF_DIR` returns nothing after Hadoop is up and running.

As per the Widrikson tutorial the following lines are added to `hadoop-env.sh`

    export JAVA_HOME=/usr/lib/jvm/jdk-8-oracle-arm32-vfp-hflt
    export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=$HADOOP_INSTALL/lib/native -Djava.net.preferIPv4Stack=true"

The following line is also encluded in the `hadoop-env.sh` file `export HADOOP_CONF_DIR=${HADOOP_CONF_DIR:-"/etc/hadoop"}` which (one would think) implies that if this file is being run then there is a variable set for `$HADOOP_CONF_DIR` however as demonstrated via `echo` this is blank. **I'm highly tempted to change this to** `export HADOOP_CONF_DIR=$HADOOP_INSTALL/etc/hadoop` **however I'll leave well enough alone**

Also if we did make the change to `$HADOOP_CONF_DIR` as suggested above I'd imagine also adding `export YARN_CONF_DIR=$HADOOP_CONF_DIR` on the next line!

This is because both Spark and Hadoop Docs state that the enviornment varibles are at a minimum `HADOOP_CONF_DIR` and `YARN_CONF_DIR` i.e.

> Ensure that HADOOP_CONF_DIR or YARN_CONF_DIR points to the directory which contains the (client side) configuration files for the Hadoop cluster.

## Running canned Hadoop Examples
Once Hadoop is up and running it would be nice to run a few simple examples to get a sense for how Hadoop runs jobs. Thankfully Hadoop comes with a series of pre-written examples to do just that.
