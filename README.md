# Installing Hadoop on single node as well multi node cluster using Debian 9 Linux VMs

> This repository reviews all required steps to install Hadoop on single node cluster as well as multi node cluster using virtual machines with Debian 9.

> **For the success of the tutorial, we assume that you already have a virtual machine equipped with Linux OS (Debian 9). This should work in principle even with other Linux distributions. You can get Virtualbox to build virtual machines. In order to concept a cluster, you must also have the ability to procure more than one virtual machine.**

The next tutorial will explain [ how to install Spark on Hadoop Yarn Multi Node Cluster][nexttuto].

[nexttuto]: https://github.com/mnassrib/installing-spark-on-hadoop-yarn-cluster
		
> # Install and Configure Hadoop with NameNode & DataNode on Single Node
		
		
## 1- Prepare Linux		       
### Commands with root	
> login as root user

``user@debian:~$ su root``

- Turnoff firewall

``root@debian:~# apt-get install firewalld``  --install firewalld if it is not installed

``root@debian:~# service firewalld status``

``root@debian:~# service firewalld stop``

``root@debian:~# systemctl disable firewalld``

- Change hostname and setup FQDN (considering a hostname as "master-node")
> Display the hostname

``root@debian:~# cat /etc/hostname``
		
> Edit the hostname

``root@debian:~# vi /etc/hostname``   --remove the existing file and write the below
	
	master-node
			
``root@debian:~# vi /etc/hosts``   --your file should look like the below

	127.0.0.1	localhost	
	192.xxx.x.1	master-node
			
> Type the following
		
``root@debian:~# hostname master-node``
		
> To check type

``root@debian:~# hostname`` --should return 
	
	master-node
	
``root@debian:~# hostname -f`` --should return 

	master-node
	
- Create user for Hadoop (considering a hadoop user as "hdpuser")
> For Debian OS users login as root and do the following:

``root@master-node:~# apt-get install sudo``

``root@master-node:~# adduser hdpuser``

``root@master-node:~# usermod -aG sudo hdpuser``  --to add a user to the sudo group. This can be done also according to (*) cited below
		
``root@master-node:~# getent group sudo``  --to verify if the new Debian sudo user was added to the group, for more details see this [site][verifsudo]. 

[verifsudo]: https://phoenixnap.com/kb/create-a-sudo-user-on-debian

``root@master-node:~# deluser --remove-home username`` --to delete username
		
> Verify Sudo Access in Debian

``root@master-node:~# su - hdpuser``  --switch to the user account you just created

``hdpuser@master-node:~$ sudo whoami``  --run any command that requires superuser access. For example, this should tell you that you are the root.

![sudowhoami](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/sudowhoami.png)

- Add Hadoop user to sudoers file (*), for more details see this [link][sudo].

[sudo]: https://www.geek17.com/fr/content/debian-9-stretch-installer-et-configurer-sudo-61

``root@master-node:~# visudo -f /etc/sudoers``  --and under the below section add
	
	## Allow root to run any commands anywhere
	root	ALL=(ALL)	All
	hdpuser ALL=(ALL)	ALL     ##add this line

### Commands with hdpuser
> login as hdpuser

- Install SSH server
		
``hdpuser@master-node:~$ sudo apt-get install ssh``
	
- Install rsync which allows remote file synchronizations using SSH

``hdpuser@master-node:~$ sudo apt-get install rsync``
	
- Generate SSH keys and setup password less SSH between Hadoop services
		
``hdpuser@master-node:~$ ssh-keygen -t rsa``  ## just press Enter for all choices

``hdpuser@master-node:~$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/id_rsa.pub/authorized_keys``

``hdpuser@master-node:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@master-node``  --(you should be able to ssh without asking for password)

``hdpuser@master-node:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@xxxxxxxx``   --(if you have more than one node, you will repeat for each node)

``hdpuser@master-node:~$ ssh hdpuser@xxxxxxxx``

	Are you sure you want to continue connecting (yes/no)? yes
	
``hdpuser@xxxxxxxx:~$ exit``
	
- Creating the needed directories:

``hdpuser@master-node:~$sudo mkdir /var/log/hadoop``

``hdpuser@master-node:~$ sudo chown -R hdpuser:hdpuser /var/log/hadoop``

``hdpuser@master-node:~$ sudo chmod -R 770 /var/log/hadoop``
		
``hdpuser@master-node:~$ sudo mkdir /bigdata``

``hdpuser@master-node:~$ sudo chown -R hdpuser:hdpuser /bigdata``

``hdpuser@master-node:~$ sudo chmod -R 770 /bigdata``


## 2- Intall JDK and Hadoop
> login as hdpuser

### Installing Java					           

- Download JDK version "[jdk-8u241-Linux-x64.tar.gz][java]", and follow installation steps:

[java]: https://www.oracle.com/java/technologies/javase-jdk8-downloads.html

``hdpuser@master-node:~$ cd /bigdata``
		
- Extract the archive to installation path, 

``hdpuser@master-node:/bigdata$ tar -xzvf jdk-8u241-Linux-x64.tar.gz``
		
- Setup Environment variables
		
``hdpuser@master-node:/bigdata$ cd ~``

``hdpuser@master-node:~$ vi .bashrc``  --add the below at the end of the file
			
	# User specific environment and startup programs
	export PATH=$HOME/.local/bin:$HOME/bin:$PATH

	# Setup JAVA Environment variables
	export JAVA_HOME=/bigdata/jdk1.8.0_241
	export PATH=$JAVA_HOME/bin:$PATH
			
``hdpuser@master-node:~$ source .bashrc`` --load the .bashrc file

- Install Java

``hdpuser@master-node:~$ sudo update-alternatives --install "/usr/bin/java" "java" "/bigdata/jdk1.8.0_241/bin/java" 0``

``hdpuser@master-node:~$ sudo update-alternatives --install "/usr/bin/javac" "javac" "/bigdata/jdk1.8.0_241/bin/javac" 0``

``hdpuser@master-node:~$ sudo update-alternatives --install "/usr/bin/javaws" "javaws" "/bigdata/jdk1.8.0_241/bin/javaws" 0``

``hdpuser@master-node:~$ sudo update-alternatives --set java /bigdata/jdk1.8.0_241/bin/java``

``hdpuser@master-node:~$ sudo update-alternatives --set javac /bigdata/jdk1.8.0_241/bin/javac``

``hdpuser@master-node:~$ sudo update-alternatives --set javaws /bigdata/jdk1.8.0_241/bin/javaws``

``hdpuser@master-node:~$ java -version``  ## to check

	hdpuser@master-node:~$ java -version
	java version "1.8.0_241"
	Java(TM) SE Runtime Environment (build 1.8.0_241-b07)
	Java HotSpot(TM) 64-Bit Server VM (build 25.241-b07, mixed mode)

			
### Installing Hadoop					     
	
- Download Hadoop archive file "[hadoop-3.1.1.tar.gz][hadoop]", and follow installation steps:

[hadoop]: https://hadoop.apache.org/releases.html

``hdpuser@master-node:~$ cd /bigdata``
		
- Extract the archive "hadoop-3.1.1.tar.gz", 
		
``hdpuser@master-node:/bigdata$ tar -zxvf hadoop-3.1.1.tar.gz``
		
- Setup Environment variables 

``hdpuser@master-node:/bigdata$ cd``  --to move to your home directory

``hdpuser@master-node:~$ vi .bashrc``  --add the following under the Java Environment Variables section into the .bashrc file
	
	# Setup Hadoop Environment variables		
	export HADOOP_HOME=/bigdata/hadoop-3.1.1
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

``hdpuser@master-node:~$ source .bashrc`` --after save the .bashrc file, load it
			
- Create directore for Hadoop Data for (NameNode & DataNode)
		
``hdpuser@master-node:~$ mkdir /bigdata/HadoopData``

``hdpuser@master-node:~$ mkdir /bigdata/HadoopData/namenode``  	*only on the NameNode server*

``hdpuser@master-node:~$ mkdir /bigdata/HadoopData/datanode``  	*on both servers*
	
- Configure Hadoop
		
``hdpuser@master-node:~$ cd $HADOOP_CONF_DIR``  ## check the environment variables you just added
	
- Modify file: **core-site.xml**
		
``hdpuser@master-node:/bigdata/hadoop-3.1.1/etc/hadoop$ vi core-site.xml``  --copy core-site.xml file
		
	<configuration>
	   <property>
		   <name>fs.defaultFS</name>
		   <value>hdfs://master-node:9000</value>
	   </property>
	</configuration>
		
- Modify file: **hdfs-site.xml**  
```diff
- The parameter "dfs.namenode.data.dir" must be kept only on the NameNode server
- If you need DataNode on the NameNode server, set the parameter "dfs.datanode.data.dir"
```

``hdpuser@master-node:/bigdata/hadoop-3.1.1/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file
		
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
		
``hdpuser@master-node:/bigdata/hadoop-3.1.1/etc/hadoop$ vi mapred-site.xml``  --copy mapred-site.xml file

	<configuration>
	   <property>
		   <name>mapreduce.framework.name</name>
		   <value>yarn</value>
	   </property>
	   <property>
		   <name>mapreduce.jobhistory.address</name>
		   <value>master-node:10020</value>
	   </property>
	   <property>
		   <name>mapreduce.jobhistory.webapp.address</name>
		   <value>master-node:19888</value>
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
		   <value>HADOOP_MAPRED_HOME=/bigdata/hadoop-3.1.1</value>
	   </property>
	   <property>
		   <name>mapreduce.map.env</name>
		   <value>HADOOP_MAPRED_HOME=/bigdata/hadoop-3.1.1</value>
	   </property>
	   <property>
		   <name>mapreduce.reduce.env</name>
		   <value>HADOOP_MAPRED_HOME=/bigdata/hadoop-3.1.1</value>
	   </property>
	   <property>
		   <name>mapreduce.application.classpath</name>
		   <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
	   </property>
	</configuration>

- Modify file: **yarn-site.xml**  
		
``hdpuser@master-node:/bigdata/hadoop-3.1.1/etc/hadoop$ vi yarn-site.xml``  --copy yarn-site.xml file
		
	<configuration>
	   <property>
		   <name>yarn.log-aggregation-enable</name>
		   <value>true</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.address</name>
		   <value>master-node:8050</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.scheduler.address</name>
		   <value>master-node:8030</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.resource-tracker.address</name>
		   <value>master-node:8025</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.admin.address</name>
		   <value>master-node:8011</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.webapp.address</name>
		   <value>master-node:8080</value>
	   </property>
	   <property>
		   <name>yarn.nodemanager.env-whitelist</name>
		   <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.webapp.https.address</name>
		   <value>master-node:8090</value>
	   </property>
	   <property>
		   <name>yarn.resourcemanager.hostname</name>
		   <value>master-node</value>
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
		   <value>hdfs://master-node:9870/tmp/hadoop-yarn</value>
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
		   <value>http://master-node:19888/jobhistory/logs</value>
	   </property>
	</configuration>
		
- Modify file: **hadoop-env.sh**       
> Edit hadoop environment file by adding the following environment variables under the section "Set Hadoop-specific environment variables here.":  
		
``hdpuser@master-node:/bigdata/hadoop-3.1.1/etc/hadoop$ vi hadoop-env.sh``  --copy hadoop-env.sh  
	
	export JAVA_HOME=/bigdata/jdk1.8.0_241
	export HADOOP_LOG_DIR=/var/log/hadoop
	export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=/bigdata/hadoop-3.1.1/lib/native"
	export HADOOP_COMMON_LIB_NATIVE_DIR=/bigdata/hadoop-3.1.1/lib/native
		
- Create **workers** file
		
``hdpuser@master-node:/bigdata/hadoop-3.1.1/etc/hadoop$ vi workers``  --copy workers file
	
	## write line for each DataNode Server
	master-node 
	
- Format the NameNode
		
``hdpuser@master-node:~$ hdfs namenode -format``
	
- Start & Stop Hadoop
###### Start
			
``hdpuser@master-node:~$ start-all.sh``
	
###### Check hadoop processes are running

	hdpuser@master-node:~$ jps  --this command should return something like
	1889 ResourceManager
	1300 NameNode
	1993 NodeManager
	2426 Jps
	1403 DataNode
	1566 SecondaryNameNode

> If you want to see the logs on the Web UI, after the application is terminated, then you need to start running the MapReduce Job History server also:

``hdpuser@master-node:~$ mr-jobhistory-daemon.sh start historyserver``

	hdpuser@master-node:~$ jps  --this command should return something like
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
| NameNode |  http://master-node:9870/ | 9870 |
| ResourceManager |    http://master-node:8080/   |   8080 |
| MapReduce JobHistory Server |    http://master-node:19888/   |   19888 |

###### Stop

``hdpuser@master-node:~$ stop-all.sh``

> # Install Hadoop with NameNode & DataNodes on Multi Nodes

In this section, we proceed to perform a multi node cluster. Only two virtual machines (nodes) will be considered. If you would a cluster composed of more than two nodes, you can apply the same steps that will be exposed below. 

Assuming that the hostnames, ip addresses and services (NameNode and/or DataNode) of the two nodes will be as follows:

| Hostname   |      IP Address     |  NameNode |  DataNode
|----------|-------------|:------:|:------:|
| master-node |  192.xxx.x.1 | &check; | &check; |
| slave-node-1 |    192.xxx.x.2   |   | &check; |

So far, we have only one machine that is ready (master-node). We have to build and configure the second server. We can clone the first machine and then modifying the necessary parameters will be a good idea.

## 1- Clone the master-node server created above
### Commands with root	
> login as root user

``hdpuser@master-node:~$ su root``

- Turnoff firewall

``root@master-node:~# service firewalld status``
		
``root@master-node:~# service firewalld stop``

``root@master-node:~# systemctl disable firewalld``
			
- Edit the hostname and setup FQDN (considering the new hostname as "slave-node-1")

``root@master-node:~# vi /etc/hostname``  --remove the existing file and write the below
			
	slave-node-1
			
``root@master-node:~# vi /etc/hosts``  --your file should look like the below
			
	127.0.0.1	localhost	
	192.xxx.x.1	master-node
	192.xxx.x.2	slave-node-1
			
- Type the following
		
``root@master-node:~# hostname slave-node-1``
		
> To check type
		
``root@master-node:~# hostname``  --should return 
	
	slave-node-1

``root@master-node:~# hostname -f``  --should return 

	slave-node-1
			
> login as hdpuser

- Generate SSH keys and setup password less SSH between Hadoop services

``hdpuser@slave-node-1:~$ ssh-keygen -t rsa``  ## just press Enter for all choices
		
``hdpuser@slave-node-1:~$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/id_rsa.pub/authorized_keys``

``hdpuser@slave-node-1:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@master-node``  (you should be able to ssh without asking for password)

``hdpuser@slave-node-1:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@slave-node-1``  (if you have more than one node, you will repeat for each node)
		
``hdpuser@slave-node-1:~$ ssh hdpuser@xxxx`` 
			
	Are you sure you want to continue connecting (yes/no)? yes

``hdpuser@xxxx:~$ exit``

- Edit the hosts file of the "master-node" server	

``hdpuser@slave-node-1:~$ ssh hdpuser@master-node``

``hdpuser@master-node:~$ vi /etc/hosts``  --your file should look like the below
			
	127.0.0.1	localhost	
	192.xxx.x.1	master-node
	192.xxx.x.2	slave-node-1

### Configure Hadoop				   
- Edit the **workers** file on the NameNode (master-node) server
		
``hdpuser@master-node:~$ vi workers``  --write line for each DataNode server (in our case both server machines are considered DataNodes)
			
	master-node  	#if you don't want this node to be DataNode, remove this line from the workers file
	slave-node-1
	
```diff 
- The most important thing here is to configure in particular the workers file on the NameNode server (master-node) because it masters the other nodes. 
- Concerning the slave-node-1 workers file, format it by leaving it empty or perform the same configuration as the master-node server workers file.
```

- Modify file: **hdfs-site.xml**  
> If you need the data to be replicated in more than one DataNode, you must modify the replication number mentioned in the hdfs-site.xml files of all the nodes. This number cannot be greater than the number of nodes.
		
>> On the NameNode & DataNode (master-node) server:

``hdpuser@master-node:/bigdata/hadoop-3.1.1/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file

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

>> On the DataNode (slave-node-1) server:

``hdpuser@slave-node-1:/bigdata/hadoop-3.1.1/etc/hadoop$ vi hdfs-site.xml``  --copy hdfs-site.xml file

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
		
- Clean up some old files on both nodes

``hdpuser@master-node:~$ rm -rf /bigdata/HadoopData/namenode/*``

``hdpuser@master-node:~$ rm -rf /bigdata/HadoopData/datanode/*``
		
``hdpuser@slave-node-1:~$ rm -rf /bigdata/HadoopData/namenode/*``

``hdpuser@slave-node-1:~$ rm -rf /bigdata/HadoopData/datanode/*``


## 2- Starting and stopping Hadoop on master-node

- Format the NameNode
		
``hdpuser@master-node:~$ hdfs namenode -format``

![format1](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/format1.png)
![format2](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/format2.png)
	
- Start Hadoop
		
###### Start
			
``hdpuser@master-node:~$ start-all.sh``

![starthadoop](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/starthadoop.PNG)

###### Check hadoop processes are running on master-node
			
	hdpuser@master-node:~$ jps 	--this command should return something like 
	4962 NodeManager
	4851 ResourceManager
	4292 NameNode
	4407 DataNode
	5320 Jps
	4590 SecondaryNameNode
		
###### Check hadoop processes are running on slave-node-1
	hdpuser@slave-node-1:~$ jps 	--this command should return something like 
	3056 Jps
	2925 NodeManager
	2815 DataNode

###### Default Web Interfaces
	NameNode	> http://master-node:9870/ 	Default HTTP port is 9870.
		
![NameNode](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/master-node9870.png)
	
	ResourceManager	> http://master-node:8080/	Default HTTP port is 8080.
		
![ResourceManager](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/master-node8080.png)

###### Get report

	hdpuser@master-node:~$ hdfs dfsadmin -report 	--this command should return something like
	Configured Capacity: 39891271680 (37.15 GB)
	Present Capacity: 25074253824 (23.35 GB)
	DFS Remaining: 25074204672 (23.35 GB)
	DFS Used: 49152 (48 KB)
	DFS Used%: 0.00%
	Replicated Blocks:
			Under replicated blocks: 0
			Blocks with corrupt replicas: 0
			Missing blocks: 0
			Missing blocks (with replication factor 1): 0
			Pending deletion blocks: 0
	Erasure Coded Block Groups:
			Low redundancy block groups: 0
			Block groups with corrupt internal blocks: 0
			Missing block groups: 0
			Pending deletion blocks: 0

	-------------------------------------------------
	Live datanodes (2):

	Name: 192.xxx.x.1:9866 (master-node)
	Hostname: master-node
	Decommission Status : Normal
	Configured Capacity: 19945635840 (18.58 GB)
	DFS Used: 24576 (24 KB)
	Non DFS Used: 6375096320 (5.94 GB)
	DFS Remaining: 12533735424 (11.67 GB)
	DFS Used%: 0.00%
	DFS Remaining%: 62.84%
	Configured Cache Capacity: 0 (0 B)
	Cache Used: 0 (0 B)
	Cache Remaining: 0 (0 B)
	Cache Used%: 100.00%
	Cache Remaining%: 0.00%
	Xceivers: 1
	Last contact: Tue Mar 31 22:46:39 CEST 2020
	Last Block Report: Tue Mar 31 22:42:43 CEST 2020
	Num of Blocks: 0


	Name: 192.xxx.x.2:9866 (slave-node-1)
	Hostname: slave-node-1
	Decommission Status : Normal
	Configured Capacity: 19945635840 (18.58 GB)
	DFS Used: 24576 (24 KB)
	Non DFS Used: 6368362496 (5.93 GB)
	DFS Remaining: 12540469248 (11.68 GB)
	DFS Used%: 0.00%
	DFS Remaining%: 62.87%
	Configured Cache Capacity: 0 (0 B)
	Cache Used: 0 (0 B)
	Cache Remaining: 0 (0 B)
	Cache Used%: 100.00%
	Cache Remaining%: 0.00%
	Xceivers: 1
	Last contact: Tue Mar 31 22:46:38 CEST 2020
	Last Block Report: Tue Mar 31 22:42:38 CEST 2020
	Num of Blocks: 0
		
###### Stop Hadoop
		
``hdpuser@master-node:~$ stop-all.sh``

![stophadoop](https://github.com/mnassrib/installing-hadoop-cluster/blob/master/images/stophadoop.png)


The next tutorial explains [how to install Spark on Hadoop Yarn Multi Node Cluster][nexttuto].

[nexttuto]: https://github.com/mnassrib/installing-spark-on-hadoop-yarn-cluster