# Update Swap on Raspberry Pi
This is a necessary step in preparing the pi hadoop cluster to run Spark. Spark cuts it pretty close on memory so we are going to update the cluster to include a larger swap file to handle the potential overflow.

I followed the [Digitalocean swapfile tutorial](https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-ubuntu-14-04) and have added a few of my own notes below. I also found the [Archlinux Wiki](https://wiki.archlinux.org/index.php/swap) very helpful. I set the file `/etc/swapfile` to have the following

- swap size 1 GB
- priority -1
- swappiness 0


## TL;DR

    # investigate the system swap
    swapon -s
    free -m

    # create a new swapfile
    sudo fallocate -l 1G /etc/swapfile

    # change permissions to root
    sudo chmod 600 /etc/swapfile
    ls -lh /etc/swapfile

    # show swap (optional turn off old swapfile)
    show -s
    sudo swapoff /var/swap

    # designate the swapfile as swap
    sudo mkswap /etc/swapfile
    sudo swapon /etc/swapfile

    # make the file permanent at reboot
    sudo nano /etc/fstab
    /etc/swapfile   none   		swap	sw		  0   	   0

    # show swappiness
    cat /proc/sys/vm/swappiness

    # change swappiness permanent at reboot
    sudo nano /etc/sysctl.conf
    vm.swappiness = 0





## Inspect the System Memory

We can check the system memory in MB using the program `free`

    pi@node1:/home/hduser $ free -m
             total       used       free     shared    buffers     cached
    Mem:           973        630        342          6         44        155
    -/+ buffers/cache:        430        542
    Swap:           99          0         99

We can also see the space available on the hard drive using

    pi@node1:/home/hduser $ df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/root        30G  6.5G   22G  24% /
    devtmpfs        483M     0  483M   0% /dev
    tmpfs           487M     0  487M   0% /dev/shm
    tmpfs           487M  6.5M  481M   2% /run
    tmpfs           5.0M  4.0K  5.0M   1% /run/lock
    tmpfs           487M     0  487M   0% /sys/fs/cgroup
    /dev/mmcblk0p1   63M   21M   43M  34% /boot

## Create a swap file
I opted for the "Faster Way" in the Digitalocean tutorial, and installed only 1 GB of memory for the swap file. There is a general rule of thumb that swap should use 2x RAM but this system is already pretty tapped out on space (if this were a 64 GB setup, I would absolustely use 2 GB for swap).

First we need to create the swapfile, I chose to put this in the `/etc` directory.

    sudo fallocate -l 1G /etc/swapfile

To verify the file is the expected size use `ls -lh /etc/swapfile`. We can then change the permissions so that only root has access to the file.

    pi@node1:/etc $ sudo chmod 600 /etc/swapfile
    pi@node1:/etc $ ls -lh swapfile
    -rw------- 1 root root 1.0G Jan 18 10:50 swapfile

The file can then be allocated as a swapfile using

    pi@node1:/etc $ sudo mkswap swapfile
    Setting up swapspace version 1, size = 1048572 KiB
    no label, UUID=019f67a1-c25f-4bc7-95ab-315a771e20ff

We then enable the swap space using `sudo swapon swapfile` which is verifed by examining the system swap. We note here that the original swap file is still intact and has a higher priority than our swapfile; since our original swap file is small (and I'm not trying to optimize this cluster) we will leave it intact but change the priority to make our larger file higher priority. (It turns out there are actually valid, and interesting, reasons to have multiple swapfiles [see here](https://unix.stackexchange.com/questions/84453/what-is-the-purpose-of-multiple-swap-files))

    pi@node1:/etc $ sudo swapon /etc/swapfile
    pi@node1:/etc $ sudo swapon -s
    Filename				Type		Size	Used	Priority
    /var/swap                              	file    	102396	0	-1
    /etc/swapfile                          	file    	1048572	0	-2
    pi@node1:/etc $ free -m
                 total       used       free     shared    buffers     cached
    Mem:           973        632        340          6         45        156
    -/+ buffers/cache:        430        542
    Swap:         1123          0       1123


If the other file is really bothersome we can turn it off using `sudo swapoff /var/swap` then remove it `sudo rm -f /var/swap` leaving the file we just created as the only swap in the system. If the file is simply turned off it is reinstated on reboot.


When the system reboots it will not automatically enable the new swap file, we can fix this by updating the bottom line of the `fstab` file with the following:

    /etc/swapfile   none   		swap	sw		  0   	   0

## Update the Swappiness
Swappiness is the percentage the system will leverage the swapfile in its normal operations. [This](https://en.wikipedia.org/wiki/Swappiness) is a nice overview of interperting the swappiness values This is useful for keeping RAM free for applications, however it comes at the cost of speed (read and write to disk is slow). For the Pi we want swappiness low. We can check the default swappiness using `cat /proc/sys/vm/swappiness`. The default value is 60 we want to change that

    pi@node1:/etc $ cat /proc/sys/vm/swappiness
    60

To update the swappiness we need to update the `sysctl.conf` file by adding the following line to the bottom of the file with

    sudo nano /etc/sysctl.conf
    vm.swappiness = 0


Note we could also have updated the file using the lines below, but this will not persist through the next reboot

    pi@node1:/etc $ sudo sysctl vm.swappiness=0
    vm.swappiness = 0


## Cache Pressure
A final related parameter is the cache pressure which will save the swap contents to cache to make lookups easier. This can be a memory drain, but since we are setting our default value to 0 I'm not going to modify this setting unless performance is aweful. This is captured in the conclusion of the Digitalocean tutorial.

The simple modification can be performed just like swappiness by adding the following line to the bottom of the `sysctl.conf` file

    vm.vfs_cache_pressure = 50
