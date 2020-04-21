# Installing Hadoop on single node as well multi-node cluster based on VMs running Debian 9 Linux

&nbsp;

> This article describes how to configure a Hadoop cluster from a pseudo-distributed configuration. The first section will explain how to install Hadoop on Debian 9 Linux. Following this installation, the Hadoop cluster consisted of only one node (i.e. single node cluster) and the MapReduce jobs will be executed in a pseudo-distributed manner. In order to use more Hadoop features, we will modify the configuration to allow jobs to be executed in a distributed manner, so the Hadoop cluster will be composed of more than on node (i.e. a multi-node cluster).


> **For the success of the tutorial, we assume that you already have a virtual machine equipped with Linux OS (Debian 9). This should work in principle even with other Linux distributions. You can get Virtualbox to build virtual machines. In order to concept a cluster, you must also have the ability to procure more than one virtual machine.**

There are several solutions, unfortunately that are generally paid solutions, to procure machines for building a cluster. If you prefer free solutions, I suggest you design your cluster by mounting Linux virtual machines on your local machine. Obviously, this later must have sufficient resources as memory and storage space. If you prefer this solution, follow these steps:

- Download [Oracle Virtualbox][virtualbox].
- Download [Linux][linux].
- [Create a Virtual Machine in Virtualbox and install Linux on it][installdebian].
- [Clone that VM][clone] after following the Hadoop installation steps.

[virtualbox]: https://www.virtualbox.org/wiki/Downloads
[linux]: https://www.debian.org/distrib/
[installdebian]: https://medium.com/platform-engineer/how-to-install-debian-linux-on-virtualbox-with-guest-additions-778afa0ee7e0
[clone]: https://protechgurus.com/clone-virtual-machine-virtualbox/

| :point_up:    | The next tutorial will explain [how to install Spark Standalone and Hadoop Yarn modes on Multi-Node Cluster][nexttuto]. |
|---------------|:------------------------|

[nexttuto]: https://github.com/mnassrib/installing-spark-standalone-and-hadoop-yarn-on-cluster

&nbsp;
&nbsp;
		
> # Install and configure Hadoop with NameNode & DataNode on single node

&nbsp;		
		
## 1- Prepare Linux		       
### Commands with root	
> login as root user

``user@debian:~$ su root``

- Turnoff firewall

``root@debian:~# service firewalld status``

``root@debian:~# service firewalld stop``

``root@debian:~# systemctl disable firewalld``

- Change hostname and setup FQDN (considering a hostname as "master-namenode")
> Display the hostname

``root@debian:~# cat /etc/hostname``
		
> Edit the hostname

``root@debian:~# vi /etc/hostname``   --remove the existing file and write the below
	
	master-namenode
			
``root@debian:~# vi /etc/hosts``   --your file should look like the below

	127.0.0.1	localhost	
	192.168.1.72	master-namenode
			
> Type the following
		
``root@debian:~# hostname master-namenode``
		
> To check type

``root@debian:~# hostname`` --should return 
	
	master-namenode
	
``root@debian:~# hostname -f`` --should return 

	master-namenode
	
- Create user for Hadoop (considering a hadoop user as "hdpuser")
> For Debian OS users login as root and do the following:

``root@master-namenode:~# apt-get install sudo``

``root@master-namenode:~# adduser hdpuser``

``root@master-namenode:~# usermod -aG sudo hdpuser``  --to add a user to the sudo group. This can be done also according to (*) cited below
		
``root@master-namenode:~# getent group sudo``  --to verify if the new Debian sudo user was added to the group, for more details see this [site][verifsudo]. 

[verifsudo]: https://phoenixnap.com/kb/create-a-sudo-user-on-debian

``root@master-namenode:~# deluser --remove-home username`` --to delete username
		
> Verify Sudo Access in Debian

``root@master-namenode:~# su - hdpuser``  --switch to the user account you just created

``hdpuser@master-namenode:~$ sudo whoami``  --run any command that requires superuser access. For example, this should tell you that you are the root.

![sudowhoami](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/sudowhoami.png)

- Add Hadoop user to sudoers file (*), for more details see this [link][sudo].

[sudo]: https://www.geek17.com/fr/content/debian-9-stretch-installer-et-configurer-sudo-61

``root@master-namenode:~# visudo -f /etc/sudoers``  --and under the below section add
	
	## Allow root to run any commands anywhere
	root	ALL=(ALL)	All
	hdpuser ALL=(ALL)	ALL     ##add this line

### Commands with hdpuser
> login as hdpuser

- Install SSH server
		
``hdpuser@master-namenode:~$ sudo apt-get install ssh``
	
- Install rsync which allows remote file synchronizations using SSH

``hdpuser@master-namenode:~$ sudo apt-get install rsync``
	
- Generate SSH keys and setup password less SSH between Hadoop services
		
``hdpuser@master-namenode:~$ ssh-keygen -t rsa``  --just press Enter for all choices

``hdpuser@master-namenode:~$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys``

``hdpuser@master-namenode:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@master-namenode``  --(you should be able to ssh without asking for password)

``hdpuser@master-namenode:~$ ssh hdpuser@master-namenode``

	Are you sure you want to continue connecting (yes/no)? yes
	
``hdpuser@master-namenode:~$ exit``
	
- Creating the needed directories:

``hdpuser@master-namenode:~$sudo mkdir /var/log/hadoop``

``hdpuser@master-namenode:~$ sudo chown -R hdpuser:hdpuser /var/log/hadoop``

``hdpuser@master-namenode:~$ sudo chmod -R 770 /var/log/hadoop``
		
``hdpuser@master-namenode:~$ sudo mkdir /bigdata``

``hdpuser@master-namenode:~$ sudo chown -R hdpuser:hdpuser /bigdata``

``hdpuser@master-namenode:~$ sudo chmod -R 770 /bigdata``

## 2- Intall JDK and Hadoop
> login as hdpuser

### Installing Java					           

- Download JDK version "[jdk-8u241-Linux-x64.tar.gz][java]", and follow installation steps:

[java]: https://www.oracle.com/java/technologies/javase-jdk8-downloads.html

``hdpuser@master-namenode:~$ cd /bigdata``
		
- Extract the archive to installation path, 

``hdpuser@master-namenode:/bigdata$ tar -xzvf jdk-8u241-Linux-x64.tar.gz``
		
- Setup Environment variables
		
``hdpuser@master-namenode:/bigdata$ cd ~``

``hdpuser@master-namenode:~$ vi .bashrc``  --add the below at the end of the file
			
	# User specific environment and startup programs
	export PATH=$HOME/.local/bin:$HOME/bin:$PATH

	# Setup JAVA Environment variables
	export JAVA_HOME=/bigdata/jdk1.8.0_241
	export PATH=$JAVA_HOME/bin:$PATH
			
``hdpuser@master-namenode:~$ source .bashrc`` --load the .bashrc file

- Install Java

``hdpuser@master-namenode:~$ sudo update-alternatives --install "/usr/bin/java" "java" "/bigdata/jdk1.8.0_241/bin/java" 0``

``hdpuser@master-namenode:~$ sudo update-alternatives --install "/usr/bin/javac" "javac" "/bigdata/jdk1.8.0_241/bin/javac" 0``

``hdpuser@master-namenode:~$ sudo update-alternatives --install "/usr/bin/javaws" "javaws" "/bigdata/jdk1.8.0_241/bin/javaws" 0``

``hdpuser@master-namenode:~$ sudo update-alternatives --set java /bigdata/jdk1.8.0_241/bin/java``

``hdpuser@master-namenode:~$ sudo update-alternatives --set javac /bigdata/jdk1.8.0_241/bin/javac``

``hdpuser@master-namenode:~$ sudo update-alternatives --set javaws /bigdata/jdk1.8.0_241/bin/javaws``

``hdpuser@master-namenode:~$ java -version``  --to check the version

	hdpuser@master-namenode:~$ java -version
	java version "1.8.0_241"
	Java(TM) SE Runtime Environment (build 1.8.0_241-b07)
	Java HotSpot(TM) 64-Bit Server VM (build 25.241-b07, mixed mode)

			
### Installing Hadoop					     
	
- Download Hadoop archive file "[hadoop-3.1.2.tar.gz][hadoop]", and follow installation steps:

[hadoop]: https://hadoop.apache.org/release.html

``hdpuser@master-namenode:~$ cd /bigdata``
		
- Extract the archive "hadoop-3.1.2.tar.gz", 
		
``hdpuser@master-namenode:/bigdata$ tar -zxvf hadoop-3.1.2.tar.gz``
		
- Setup Environment variables 

``hdpuser@master-namenode:/bigdata$ cd``  --to move to your home directory

``hdpuser@master-namenode:~$ vi .bashrc``  --add the following under the Java Environment variables section into the .bashrc file
	
	# Setup Hadoop Environment variables		
	export HADOOP_HOME=/bigdata/hadoop-3.1.2
	export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
	export HADOOP_NAMENODE_OPTS="-XX:+UseParallelGC"
	export HADOOP_MAPRED_HOME=$HADOOP_HOME
	export HADOOP_HDFS_HOME=$HADOOP_HOME
	export HADOOP_COMMON_HOME=$HADOOP_HOME
	export HADOOP_YARN_HOME=$HADOOP_HOME
	export YARN_HOME=$HADOOP_HOME
	export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
	export HADOOP_CLASSPATH=$JAVA_HOME/lib/tools.jar
	export HADOOP_LOG_DIR=/var/log/hadoop
	export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=$HADOOP_HOME/lib/native"
	export PATH=$HOME/.local/bin:$HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$PATH

	export HADOOP_CLASSPATH=$HADOOP_CONF_DIR:$HADOOP_COMMON_HOME/*:$HADOOP_COMMON_HOME/lib/*:$HADOOP_HDFS_HOME/*:$HADOOP_HDFS_HOME/lib/*:$HADOOP_MAPRED_HOME/*:$HADOOP_MAPRED_HOME/lib/*:$HADOOP_YARN_HOME/*:$HADOOP_YARN_HOME/lib/*:$HADOOP_CLASSPATH
	
	# Control Hadoop
	alias Start_HADOOP='$HADOOP_HOME/sbin/start-all.sh;mapred --daemon start historyserver'
	alias Stop_HADOOP='$HADOOP_HOME/sbin/stop-all.sh;mapred --daemon stop historyserver'


``hdpuser@master-namenode:~$ source .bashrc`` --after save the .bashrc file, load it
			
- Create directore for Hadoop Data for (NameNode & DataNode)
		
``hdpuser@master-namenode:~$ mkdir /bigdata/HadoopData``

``hdpuser@master-namenode:~$ mkdir /bigdata/HadoopData/namenode``  	--*only on the NameNode server*

``hdpuser@master-namenode:~$ mkdir /bigdata/HadoopData/datanode``  	--*on all the the DataNodes servers*
	
- Configure Hadoop
		
``hdpuser@master-namenode:~$ cd $HADOOP_CONF_DIR``  --check the environment variables you just added
	
- Modify file: **core-site.xml**
		
``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi core-site.xml``  --copy core-site.xml file
		
	<configuration>
	   <property>
		   <name>fs.defaultFS</name>
		   <value>hdfs://master-namenode:9000</value>
	   </property>
	</configuration>
		
- Modify file: **hdfs-site.xml**  

| :exclamation: | The parameter ``dfs.namenode.data.dir`` must be kept only on the NameNode server. If you need DataNode on the NameNode server, set the parameter ``dfs.datanode.data.dir``       |
|---------------|:------------------------|

``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file
		
	<configuration>
	   <property>
		   <name>dfs.namenode.name.dir</name>
		   <value>file:///bigdata/HadoopData/namenode</value>
	   </property>
	   <property>
		   <name>dfs.datanode.data.dir</name>
		   <value>file:///bigdata/HadoopData/datanode</value>
	   </property>
	   <property>
		   <name>dfs.blocksize</name>
		   <value>134217728</value>
	   </property>
	   <property>
		   <name>dfs.replication</name>
		   <value>1</value>
	   </property>
	   <property>
		   <name>dfs.permissions</name>
		   <value>false</value>
	   </property>
	</configuration>

- Modify file: **mapred-site.xml**
		
``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi mapred-site.xml``  --copy mapred-site.xml file

	<configuration>
	   <property>
		   <name>mapreduce.framework.name</name>
		   <value>yarn</value>
	   </property>
	   <property>
		   <name>mapreduce.jobhistory.address</name>
		   <value>master-namenode:10020</value>
	   </property>
	   <property>
		   <name>mapreduce.jobhistory.webapp.address</name>
		   <value>master-namenode:19888</value>
	   </property>
	   <property>
		   <name>mapreduce.jobhistory.intermediate-done-dir</name>
		   <value>var/log/hadoop/tmp</value>
	   </property>
	   <property>
		   <name>mapreduce.jobhistory.done-dir</name>
		   <value>var/log/hadoop/done</value>
	   </property>
	   <property>
		   <name>mapreduce.map.memory.mb</name>
		   <value>512</value>
	   </property>
	   <property>
		   <name>mapreduce.reduce.memory.mb</name>
		   <value>512</value>
	   </property>
	   <property>
		   <name>mapreduce.map.java.opts</name>
		   <value>-Xmx512M</value>
	   </property>
	   <property>
		   <name>mapreduce.job.maps</name>
		   <value>2</value>
	   </property>
	   <property>
		   <name>mapreduce.reduce.java.opts</name>
		   <value>-Xmx512M</value>
	   </property>
	   <property>
		   <name>mapreduce.task.io.sort.mb</name>
		   <value>128</value>
	   </property>
	   <property>
		   <name>mapreduce.task.io.sort.factor</name>
		   <value>15</value>
	   </property>
	   <property>
		   <name>mapreduce.reduce.shuffle.parallelcopies</name>
		   <value>2</value>
	   </property>
	   <property>
		   <name>yarn.app.mapreduce.am.env</name>
		   <value>HADOOP_MAPRED_HOME=/bigdata/hadoop-3.1.2</value>
	   </property>
	   <property>
		   <name>mapreduce.map.env</name>
		   <value>HADOOP_MAPRED_HOME=/bigdata/hadoop-3.1.2</value>
	   </property>
	   <property>
		   <name>mapreduce.reduce.env</name>
		   <value>HADOOP_MAPRED_HOME=/bigdata/hadoop-3.1.2</value>
	   </property>
	   <property>
		   <name>mapreduce.application.classpath</name>
		   <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
	   </property>
	</configuration>

- Modify file: **yarn-site.xml**  
		
``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi yarn-site.xml``  --copy yarn-site.xml file
		
	<configuration>
	   <property>
		   <name>yarn.log-aggregation-enable</name>
		   <value>true</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.address</name>
		   <value>master-namenode:8050</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.scheduler.address</name>
		   <value>master-namenode:8030</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.resource-tracker.address</name>
		   <value>master-namenode:8025</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.admin.address</name>
		   <value>master-namenode:8011</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.webapp.address</name>
		   <value>master-namenode:8080</value>
	   </property>
	   <property>
		   <name>yarn.nodemanager.env-whitelist</name>
		   <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.webapp.https.address</name>
		   <value>master-namenode:8090</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.hostname</name>
		   <value>master-namenode</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.scheduler.class</name>
		   <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
	   </property>
	   <property>
		   <name>yarn.nodemanager.local-dirs</name>
		   <value>file:///var/log/hadoop</value>
	   </property>
	   <property>
		   <name>yarn.nodemanager.log-dirs</name>
		   <value>file:///var/log/hadoop</value>
	   </property>
	   <property>
		   <name>yarn.nodemanager.remote-app-log-dir</name>
		   <value>hdfs://master-namenode:9870/tmp/hadoop-yarn</value>
	   </property>
	   <property>
		   <name>yarn.nodemanager.remote-app-log-dir-suffix</name>
		   <value>logs</value>
	   </property>
	   <property>
		   <name>yarn.nodemanager.aux-services</name>
		   <value>mapreduce_shuffle</value>
	   </property>
	   <property>
		   <name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>  
		   <value>org.apache.hadoop.mapred.ShuffleHandler</value>
	   </property>
	   <property>
		   <name>yarn.log.server.url</name>
		   <value>http://master-namenode:19888/jobhistory/logs</value>
	   </property>
	</configuration>
		
- Modify file: **hadoop-env.sh**       

| :memo:        | Edit Hadoop environment file by adding the following environment variables under the section "Set Hadoop-specific environment variables here.":       |
|---------------|:------------------------|
		
``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hadoop-env.sh``  --copy hadoop-env.sh  
	
	export JAVA_HOME=/bigdata/jdk1.8.0_241
	export HADOOP_LOG_DIR=/var/log/hadoop
	export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=/bigdata/hadoop-3.1.2/lib/native"
	export HADOOP_COMMON_LIB_NATIVE_DIR=/bigdata/hadoop-3.1.2/lib/native
		
- Create **workers** file
		
``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi workers``  --copy workers file
| :exclamation: | Write line for each DataNode server       |
|---------------|:------------------------|

	master-namenode 
	
- Format the NameNode
		
``hdpuser@master-namenode:~$ hdfs namenode -format``
	
- Start & Stop Hadoop

| :point_up:    | In principle to start Hadoop, we only need to type ``start-all.sh``. However, I created two aliases ``Start_HADOOP`` and ``Stop_HADOOP`` into the environment variables that will ensure the execution of Hadoop. I created these aliases in order to avoid conflicts with some commands existing with Spark which will be soon installed on the same machines. The same rules will also be applied with Spark. Once an application is terminated, you need to start running the MapReduce Job History Server if you want to see the logs on the Web UI. For this, I added ``mapred --daemon start historyserver`` and ``mapred --daemon stop historyserver`` commands into the two created aliases. |
|---------------|:------------------------|

###### Start
			
``hdpuser@master-namenode:~$ Start_HADOOP``
	
###### Check Hadoop processes are running

	hdpuser@master-namenode:~$ jps  --this command should return something like
	1889 ResourceManager
	1300 NameNode
	1093 JobHistoryServer
	1993 NodeManager
	2426 Jps
	1403 DataNode
	1566 SecondaryNameNode
			
###### Default Web Interfaces
	
| Service   |      Address web      |  Default HTTP port |
|-----------|-----------------------|-------------------:|
| NameNode |  http://master-namenode:9870/ | 9870 |
| ResourceManager |    http://master-namenode:8080/   |   8080 |
| MapReduce JobHistory Server |    http://master-namenode:19888/   |   19888 |

###### Stop

``hdpuser@master-namenode:~$ Stop_HADOOP``

&nbsp;
&nbsp;

> # Install Hadoop with NameNode & DataNodes on multi-nodes

&nbsp;

In this second section, we proceed to perform a multi-node cluster. Three virtual machines (nodes) will be considered. If you would a cluster composed of more than three nodes, you can apply the same steps that will be exposed below. 

Assuming the hostnames, ip addresses and services (NameNode and/or DataNode) of the three nodes will be as follows:

| Hostname   |      IP Address     |  NameNode |  DataNode
|----------|-------------|:------:|:------:|
| master-namenode |  192.168.1.72 | &check; | &check; |
| slave-datanode-1 |    192.168.1.73   |   | &check; |
| slave-datanode-2 |    192.168.1.74   |   | &check; |

So far, we have only one machine (master-namenode) that is ready. We have to build and configure the two other added machines. We can clone the master-namenode machine twice, then changing the necessary parameters seems like a good idea.

## 1- Clone twice the master-namenode server created above
### Commands with root	
> login as root user on the two cloned machines 

``hdpuser@master-namenode:~$ su root``
			
- Edit the hostname and setup FQDN (considering the new hostnames as "slave-datanode-1" and "slave-datanode-2")

> On the first cloned machine (slave-datanode-1 server)

``root@master-namenode:~# vi /etc/hostname``  --remove the existing file and write the below
			
	slave-datanode-1
			
``root@master-namenode:~# vi /etc/hosts``  --your file should look like the below
			
	127.0.0.1	localhost	
	192.168.1.72	master-namenode
	192.168.1.73	slave-datanode-1
	192.168.1.74	slave-datanode-2

``root@master-namenode:~# hostname slave-datanode-1``

``root@master-namenode:~# hostname``  --should return 
	
	slave-datanode-1

``root@master-namenode:~# hostname -f``  --should return 

	slave-datanode-1
	
> On the second cloned machine (slave-datanode-2 server)

``root@master-namenode:~# vi /etc/hostname``  --remove the existing file and write the below
			
	slave-datanode-2
			
``root@master-namenode:~# vi /etc/hosts``  --your file should look like the below
			
	127.0.0.1	localhost	
	192.168.1.72	master-namenode
	192.168.1.73	slave-datanode-1
	192.168.1.74	slave-datanode-2

``root@master-namenode:~# hostname slave-datanode-2``

``root@master-namenode:~# hostname``  --should return 
	
	slave-datanode-2

``root@master-namenode:~# hostname -f``  --should return 

	slave-datanode-2
			
> login as hdpuser on "master-namenode" server

- Edit the hosts file into the "master-namenode" server	

``hdpuser@master-namenode:~$ vi /etc/hosts``  --your file should look like the below
			
	127.0.0.1	localhost	
	192.168.1.72	master-namenode
	192.168.1.73	slave-datanode-1
	192.168.1.74	slave-datanode-2

- Setup password less SSH between Hadoop services

``hdpuser@master-namenode:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@slave-datanode-1``  

``hdpuser@master-namenode:~$ ssh hdpuser@slave-datanode-1``  

	Are you sure you want to continue connecting (yes/no)? yes

``hdpuser@slave-datanode-1:~$ exit``

``hdpuser@master-namenode:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@slave-datanode-2``  

``hdpuser@master-namenode:~$ ssh hdpuser@slave-datanode-2``  

	Are you sure you want to continue connecting (yes/no)? yes

``hdpuser@slave-datanode-2:~$ exit``
		
### Configure Hadoop				   
- Edit the **workers** file on the NameNode (master-namenode) server

| :warning: WARNING          |
|:---------------------------|
| The goal here is to configure in particular the workers file into the NameNode server (master-namenode). Since this later orchestrates all the DataNode servers, it must know their hostnames by mentioning them in its workers file. About the workers files into the slave-datanode-1 and slave-datanode-2 servers, format by leaving them empty.     |

``hdpuser@master-namenode:~$ vi workers``  --write line for each DataNode server (in our case all the machines are considered as DataNodes)
			
	master-namenode   #remove this line from the workers file if you don't want this node to be DataNode
	slave-datanode-1
	slave-datanode-2
	
- Modify file: **hdfs-site.xml**  

| :warning: WARNING          |
|:---------------------------|
| If you need the data to be replicated in more than one DataNode, you must modify the replication number mentioned in the **hdfs-site.xml** files on all the nodes. This number cannot be greater than the number of nodes. We're going to set it here at 2.     |
		
> On the NameNode & DataNode (master-namenode) server:

``hdpuser@master-namenode:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file

	<configuration>
	   <property>
		   <name>dfs.namenode.name.dir</name>
		   <value>file:///bigdata/HadoopData/namenode</value>
	   </property>
	   <property>
		   <name>dfs.datanode.data.dir</name>
		   <value>file:///bigdata/HadoopData/datanode</value>
	   </property>
	   <property>
		   <name>dfs.blocksize</name>
		   <value>134217728</value>
	   </property>
	   <property>
		   <name>dfs.replication</name>
		   <value>2</value>
	   </property>
	   <property>
		   <name>dfs.permissions</name>
		   <value>false</value>
	   </property>
	</configuration>

> On the DataNodes (slave-datanode-1 and slave-datanode-2) servers:

``hdpuser@slave-datanode-1:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file

	<configuration>
	   <property>
		   <name>dfs.datanode.data.dir</name>
		   <value>file:///bigdata/HadoopData/datanode</value>
	   </property>
	   <property>
		   <name>dfs.blocksize</name>
		   <value>134217728</value>
	   </property>
	   <property>
		   <name>dfs.replication</name>
		   <value>2</value>
	   </property>
	   <property>
		   <name>dfs.permissions</name>
		   <value>false</value>
	   </property>
	</configuration>

``hdpuser@slave-datanode-2:/bigdata/hadoop-3.1.2/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file

	<configuration>
	   <property>
		   <name>dfs.datanode.data.dir</name>
		   <value>file:///bigdata/HadoopData/datanode</value>
	   </property>
	   <property>
		   <name>dfs.blocksize</name>
		   <value>134217728</value>
	   </property>
	   <property>
		   <name>dfs.replication</name>
		   <value>2</value>
	   </property>
	   <property>
		   <name>dfs.permissions</name>
		   <value>false</value>
	   </property>
	</configuration>
		
- Clean up some old files on all the nodes

``hdpuser@master-namenode:~$ rm -rf /bigdata/HadoopData/namenode/*``

``hdpuser@master-namenode:~$ rm -rf /bigdata/HadoopData/datanode/*``
		
``hdpuser@slave-datanode-1:~$ rm -rf /bigdata/HadoopData/datanode/*``

``hdpuser@slave-datanode-2:~$ rm -rf /bigdata/HadoopData/datanode/*``


## 2- Starting and stopping Hadoop on master-namenode

- Format the NameNode
		
``hdpuser@master-namenode:~$ hdfs namenode -format``

![format1](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/format1.png)
![format2](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/format2.png)
	
- Start Hadoop
		
###### Start
			
``hdpuser@master-namenode:~$ Start_HADOOP``

![starthadoop](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/starthadoop.png)

###### Check Hadoop processes are running on master-namenode
			
``hdpuser@master-namenode:~$ jps``

![namenodejps](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/namenodejps.png)
		
###### Check Hadoop processes are running on slave-datanode-1

``hdpuser@master-datanode-1:~$ jps``

![datanode1jps](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/datanode1jps.png)

###### Check Hadoop processes are running on slave-datanode-2

``hdpuser@master-datanode-2:~$ jps``

![datanode2jps](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/datanode2jps.png)

###### Default Web Interfaces
> NameNode: http://master-namenode:9870/ 
		
![NameNode](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/master-node9870.png)
	
> ResourceManager: http://master-namenode:8080/
		
![ResourceManager](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/master-node8080.png)

###### Get report

	hdpuser@master-namenode:~$ hdfs dfsadmin -report 	--this command should return something like
	Configured Capacity: 59836907520 (55.73 GB)
	Present Capacity: 27630944256 (25.73 GB)
	DFS Remaining: 27630858240 (25.73 GB)
	DFS Used: 86016 (84 KB)
	DFS Used%: 0.00%
	Replicated Blocks:
			Under replicated blocks: 0
			Blocks with corrupt replicas: 0
			Missing blocks: 0
			Missing blocks (with replication factor 1): 0
			Low redundancy blocks with highest priority to recover: 0
			Pending deletion blocks: 0
	Erasure Coded Block Groups:
			Low redundancy block groups: 0
			Block groups with corrupt internal blocks: 0
			Missing block groups: 0
			Low redundancy blocks with highest priority to recover: 0
			Pending deletion blocks: 0

	-------------------------------------------------
	Live datanodes (3):

	Name: 192.168.1.72:9866 (master-namenode)
	Hostname: master-namenode
	Decommission Status : Normal
	Configured Capacity: 19945635840 (18.58 GB)
	DFS Used: 28672 (28 KB)
	Non DFS Used: 9707601920 (9.04 GB)
	DFS Remaining: 9201225728 (8.57 GB)
	DFS Used%: 0.00%
	DFS Remaining%: 46.13%
	Configured Cache Capacity: 0 (0 B)
	Cache Used: 0 (0 B)
	Cache Remaining: 0 (0 B)
	Cache Used%: 100.00%
	Cache Remaining%: 0.00%
	Xceivers: 1
	Last contact: Wed Apr 15 16:44:05 CEST 2020
	Last Block Report: Wed Apr 15 16:42:00 CEST 2020
	Num of Blocks: 0


	Name: 192.168.1.73:9866 (slave-datanode-1)
	Hostname: slave-datanode-1
	Decommission Status : Normal
	Configured Capacity: 19945635840 (18.58 GB)
	DFS Used: 28672 (28 KB)
	Non DFS Used: 9695444992 (9.03 GB)
	DFS Remaining: 9213382656 (8.58 GB)
	DFS Used%: 0.00%
	DFS Remaining%: 46.19%
	Configured Cache Capacity: 0 (0 B)
	Cache Used: 0 (0 B)
	Cache Remaining: 0 (0 B)
	Cache Used%: 100.00%
	Cache Remaining%: 0.00%
	Xceivers: 1
	Last contact: Wed Apr 15 16:44:04 CEST 2020
	Last Block Report: Wed Apr 15 16:41:56 CEST 2020
	Num of Blocks: 0


	Name: 192.168.1.74:9866 (slave-datanode-2)
	Hostname: slave-datanode-2
	Decommission Status : Normal
	Configured Capacity: 19945635840 (18.58 GB)
	DFS Used: 28672 (28 KB)
	Non DFS Used: 9692577792 (9.03 GB)
	DFS Remaining: 9216249856 (8.58 GB)
	DFS Used%: 0.00%
	DFS Remaining%: 46.21%
	Configured Cache Capacity: 0 (0 B)
	Cache Used: 0 (0 B)
	Cache Remaining: 0 (0 B)
	Cache Used%: 100.00%
	Cache Remaining%: 0.00%
	Xceivers: 1
	Last contact: Wed Apr 15 16:44:04 CEST 2020
	Last Block Report: Wed Apr 15 16:41:56 CEST 2020
	Num of Blocks: 0

		
###### Stop Hadoop
		
``hdpuser@master-namenode:~$ Stop_HADOOP``

![stophadoop](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/stophadoop.png)

&nbsp;

| :point_up:    | The next tutorial explains [how to install Spark Standalone and Hadoop Yarn modes on Multi-Node Cluster][nexttuto]. |
|---------------|:------------------------|

[nexttuto]: https://github.com/mnassrib/installing-spark-standalone-and-hadoop-yarn-on-cluster