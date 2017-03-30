# Hadoop Python Streaming Example
This project is one of my first steps in building up an understanding and basic compentency in Hadoop. Although there are many tools available to make working with Hadoop and MapReduce easier, i.e. to short-cut writing programs in java, I wanted to try and work through some simple examples of how the fundamental Hadoop echo-system works... the only problem is I don't program in Java. This is where streaming comes in. Hadoop allows other programs i.e. Python, Ruby, C to interact with Map and Reduce (without compiling code to Java e.g. Jython). This functionality is called Streaming and the interface is described by the [Streaming API](http://hadoop.apache.org/docs/r2.7.3/hadoop-streaming/HadoopStreaming.html). Ultimately, the goal of this simple example is to write the conanical WordCount program in Python and is informed by some online tutorials by [Michael Noll](http://www.michael-noll.com/tutorials/writing-an-hadoop-mapreduce-program-in-python/), [Glenn Klockwood](http://www.glennklockwood.com/data-intensive/hadoop/streaming.html) and [Tutorials Point](https://www.tutorialspoint.com/hadoop/hadoop_streaming.htm).


## Preparing HDFS
Hadoop is built on several sub programs that handle the distributed file system and distributed processing. The Hadoop Distributed File System (HDFS) is essentially a program that handles the distribution of data across the cluster. Data must be imported into HDFS such that Hadoop can setup files for processing.

### Loading a Sample File into HDFS
I'm going to use a sample file from a tutorial I used to get Hadoop setup on the Raspberry pi initially ([described here](http://www.widriksson.com/raspberry-pi-2-hadoop-2-cluster/#Run-hadoop-hello-world-wordcount)). First we download the files.

    cd ~
    wget http://www.widriksson.com/wp-content/uploads/2014/10/hadoop_sample_txtfiles.tar.gz
    tar -xvzf hadoop_sample_txtfiles.tar.gz ~/

Then we need to load them into HDFS

    hadoop fs -put smallfile.txt /smallfile.txt
    hadoop fs -put mediumfile.txt /mediumfile.txt

We can then check the contents of HDFS, where we expect to see the two new directories

    pi@raspberrypi:/home/hduser $ hadoop fs -ls /
    Found 3 items
    -rw-r--r--   1 hduser supergroup   35926176 2016-12-11 22:32 /mediumfile.txt
    -rw-r--r--   1 hduser supergroup    1226438 2016-12-11 22:31 /smallfile.txt
    drwx------   - hduser supergroup          0 2016-12-11 22:37 /tmp



## Simple Python Code
As the simplest example of the WordCount program we will use the **mapper.py** and **reducer.py** code from Michael Noll. It is helpful to note that he also provides an implementation using Iterators and Generators, however the example below is extremely straight forward and has the added benefit that we can run simple tests in the terminal.

**mapper.py**

    #!/usr/bin/env python

    import sys

    # input comes from STDIN (standard input)
    for line in sys.stdin:
        # remove leading and trailing whitespace
        line = line.strip()
        # split the line into words
        words = line.split()
        # increase counters
        for word in words:
            # write the results to STDOUT (standard output);
            # what we output here will be the input for the
            # Reduce step, i.e. the input for reducer.py
            #
            # tab-delimited; the trivial word count is 1
            print '%s\t%s' % (word, 1)


**reducer.py**

    #!/usr/bin/env python

    from operator import itemgetter
    import sys

    current_word = None
    current_count = 0
    word = None

    # input comes from STDIN
    for line in sys.stdin:
    # remove leading and trailing whitespace
    line = line.strip()

    # parse the input we got from mapper.py
    word, count = line.split('\t', 1)

    # convert count (currently a string) to int
    try:
        count = int(count)
    except ValueError:
        # count was not a number, so silently
        # ignore/discard this line
        continue

    # this IF-switch only works because Hadoop sorts map output
    # by key (here: word) before it is passed to the reducer
    if current_word == word:
        current_count += count
    else:
        if current_word:
            # write result to STDOUT
            print '%s\t%s' % (current_word, current_count)
        current_count = count
        current_word = word

    # do not forget to output the last word if needed!
    if current_word == word:
    print '%s\t%s' % (current_word, current_count)

As per Noll's excellent tutorial we can also test that our functions work as intended as follows:

    pi@raspberrypi:/home/hduser $ echo "foo foo quux labs foo bar quux" | /home/hduser/mapper.py
    foo	1
    foo	1
    quux	1
    labs	1
    foo	1
    bar	1
    quux	1

    pi@raspberrypi:/home/hduser $ echo "foo foo quux labs foo bar quux" | /home/hduser/mapper.py | sort -k1,1 | /home/hduser/reducer.py
    bar	1
    foo	3
    labs	1
    quux	2

    pi@raspberrypi:/home/hduser $ cat smallfile.txt | /home/hduser/mapper.py | less

## Running the Python Stream
It is important to note that both tutorials I've used for reference are for significantly older versions of Hadoop, the Streaming API for 2.7.3 is found [here](http://hadoop.apache.org/docs/r2.7.3/hadoop-streaming/HadoopStreaming.html). **The basic syntax for streaming is as follows:**

    hadoop jar hadoop-streaming-2.7.3.jar \
      -input myInputDirs \
      -output myOutputDir \
      -mapper /bin/cat \
      -reducer /usr/bin/wc

To run our specific job we will use (note we should be logged in as hduser to run this job, i.e. > su hduser)

    cd /home/pi/hadoop-2.7.3-src/hadoop-tools/hadoop-streaming/target/
    hadoop jar ./hadoop-streaming-2.7.3.jar  \
      -input /smallfile.txt \
      -output /small-python-out \
      -mapper /home/hduser/mapper.py \
      -reducer /home/hduser/reducer.py

we can check the status HDFS through our browser using http://localhost:8088. Since we are running headless the localhost is the wireless address of our pi or 192.168.1.166

    http://192.168.1.166:8088/cluster (HDFS Browser interface)


we can also check the output of our job, found in /small-python-out/ using the following. It is helpful to note that part-00000 and part-00001 represent the first and second reducer tasks [as explained here](http://stackoverflow.com/questions/10666488/what-are-success-and-part-r-00000-files-in-hadoop)

    hduser@raspberrypi:/home/pi/ $ hadoop fs -ls /small-python-out/
    Found 3 items
    -rw-r--r--   1 hduser supergroup          0 2016-12-17 13:59 /small-python-out/_SUCCESS
    -rw-r--r--   1 hduser supergroup     105919 2016-12-17 13:59 /small-python-out/part-00000
    -rw-r--r--   1 hduser supergroup     106768 2016-12-17 13:59 /small-python-out/part-00001

Alternatively, we could view our HDFS filesystem through through the browser based interface to the cluster http://localhost:50070  Then click > Utilities > Browse the Filesystem > small-python-out to see the results of the Streaming job.

    http://192.168.1.166:50070 (Application Browser interface)


we can see our output from part-00000 using the following command. *note that for some unknown reason we produced 2 reduce files 00000 and 00001 - I suspect this is the result of a 5MB block size where our total output ~ 10 MB. For context, our streaming output exactly matches the non-streaming WordCount example found in [this tutorial](http://www.widriksson.com/raspberry-pi-2-hadoop-2-cluster/)*

    hduser@raspberrypi:/home/pi/ $ hadoop fs -cat /small-python-out/part-00000 | less

## Retrieving the HFDS Output
Since some portion of this example is aimed at highlighting useful everyday commands in Hadoop, lets pull our final result out of HDFS and into our user space using the **get** command

    hduser@raspberrypi:~ $ hadoop fs -get /small-python-out/ /home/hduser/small-python-out
    hduser@raspberrypi:~ $ tree ./small-python-out/ -L 2
    ./small-python-out/
    ├── part-00000
    ├── part-00001
    └── _SUCCESS



## Python WordCount Iterators and Generators
While the previous example is nice and simple we can benefit from the use of iterators and generators to make the python code more memory efficient. Michael Noll provides the following updates to **mapper.py** and **reducer.py** as shown below:

**mapper.py**

    #!/usr/bin/env python
    """A more advanced Mapper, using Python iterators and generators."""

    import sys

    def read_input(file):
        for line in file:
            # split the line into words
            yield line.split()

    def main(separator='\t'):
        # input comes from STDIN (standard input)
        data = read_input(sys.stdin)
        for words in data:
            # write the results to STDOUT (standard output);
            # what we output here will be the input for the
            # Reduce step, i.e. the input for reducer.py
            #
            # tab-delimited; the trivial word count is 1
            for word in words:
                print '%s%s%d' % (word, separator, 1)

    if __name__ == "__main__":
        main()

**reducer.py**

    #!/usr/bin/env python
    """A more advanced Reducer, using Python iterators and generators."""

    from itertools import groupby
    from operator import itemgetter
    import sys

    def read_mapper_output(file, separator='\t'):
        for line in file:
            yield line.rstrip().split(separator, 1)

    def main(separator='\t'):
        # input comes from STDIN (standard input)
        data = read_mapper_output(sys.stdin, separator=separator)
        # groupby groups multiple word-count pairs by word,
        # and creates an iterator that returns consecutive keys and their group:
        #   current_word - string containing a word (the key)
        #   group - iterator yielding all ["&lt;current_word&gt;", "&lt;count&gt;"] items
        for current_word, group in groupby(data, itemgetter(0)):
            try:
                total_count = sum(int(count) for current_word, count in group)
                print "%s%s%d" % (current_word, separator, total_count)
            except ValueError:
                # count was not a number, so silently discard this item
                pass

    if __name__ == "__main__":
        main()
