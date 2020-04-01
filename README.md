# Installing Hadoop 3 on single node and multi-node clusters of Debian 9 VMs
> This repository describes all necessary steps to install Hadoop on single node cluster as well as multi node cluster of virtual machines with Debian 9.

> **For the success of this tutorial, we assume that you already have a virtual machine equipped with Linux OS (Debian 9). This should work in principle even with other Linux distributions. You can get Virtualbox to build virtual machines. To design a cluster, you must also have the ability to procure more than one virtual machine.**

		#################################################################################
		##	Install and Configure Hadoop with NameNode & DataNode on Single Node   ##
		#################################################################################
		
		
## 1- Prepare Linux		       
### Commands with root	
> login as root user

	## Turnoff firewall
		apt-get install firewalld  --install firewalld if it is not installed
		service firewalld status
		service firewalld stop
		systemctl disable firewalld
	
	## Change hostname and setup FQDN (considering a hostname as "master-node")
		# Display the hostname
		cat /etc/hostname
		
		# Edit the hostname
		vi /etc/hostname   --remove the existing file and write the below
			master-node
			
		vi /etc/hosts   --your file should look like the below
			127.0.0.1		localhost	
			192.168.1.5		master-node
			
		# Type the following
		hostname master-node
		
		# To check type
		hostname
			should return -> master-node
		hostname -f
			should return -> master-node
	
	## Create user for HADOOP (considering a hadoop user as "hdpuser")
		For Debian OS users login as root and do the following:
		apt-get install sudo
		adduser hdpuser
		usermod -aG sudo hdpuser  --to add a user to the sudo group. This can be done also according to (*)
		getent group sudo  --to verify the new Debian sudo user was added to the group, for more details see https://phoenixnap.com/kb/create-a-sudo-user-on-debian
		deluser --remove-home username --To delete user
		
		## Verify Sudo Access in Debian
			su - hdpuser  --switch to the user account you just created
			sudo whoami  --run any command that requires superuser access. For example, this should tell you that you are the root.

	## Add Hadoop user to sudoers file (*), for more details see https://www.geek17.com/fr/content/debian-9-stretch-installer-et-configurer-sudo-61
		## visudo -f /etc/sudoers
		
		and under the below section
			## Allow root to run any commands anywhere
			root	ALL=(ALL)	All
			hdpuser ALL=(ALL)	ALL     ##add this line

### Commands with hdpuser
> login as hdpuser

	## Install SSH server
		sudo apt-get install ssh
	
	## Install rsync which allows remote file synchronizations using SSH
		sudo apt-get install rsync
	
	## Generate SSH keys and setup password less SSH between Hadoop services
		sudo ssh-keygen -t rsa  ## just press Enter for all choices
		cat ~/.ssh/id_rsa.pub >> ~/.ssh/id_rsa.pub/authorized_keys
		ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@master-node (you should be able to ssh without asking for password)
		ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@xxxxxxxx (if you have more than one node, you will repeat for each node)
		ssh hdpuser@xxxx 
			- yes
		logout or exit
	
	## Creating the needed directories:
		sudo mkdir /var/log/hadoop
		sudo chown -R hdpuser:hdpuser /var/log/hadoop
		sudo chmod -R 770 /var/log/hadoop
		
		sudo mkdir /bigdata
		sudo chown -R hdpuser:hdpuser /bigdata
		sudo chmod -R 770 /bigdata


## 2- Intall JDK and Hadoop
> login as hdpuser

### Installing Java					           

	## Download JDK version "jdk-8u191-Linux-x64.tar.gz", and follow installation steps:
		cd /bigdata
		
	## Extract the archive to installation path, 
		tar -xzvf jdk-8u191-Linux-x64.tar.gz -C /bigdata
		
	## Setup Environment variables
		cd ~
		vi .bash_profile
		## Add the below at the end of file
			export JAVA_HOME=/bigdata/jdk1.8.0_191
			export PATH=$PATH:$JAVA_HOME/bin
			
		--after save the bash_profile load it
		source .bash_profile
		
			The .bash_profile file should like this
			---------------------------------------
			# .bash_profile

			# Get the aliases and functions
			if [ -f ~/.bashrc ]; then
					. ~/.bashrc
			fi

			# User specific environment and startup programs

			export PATH=$PATH:$HOME/.local/bin:$HOME/bin

			# Setup JAVA Environment variables
			export JAVA_HOME=/bigdata/jdk1.8.0_191
			export PATH=$PATH:$JAVA_HOME/bin
			---------------------------------------

		## Install Java
		sudo update-alternatives --install "/usr/bin/java" "java" "/bigdata/jdk1.8.0_191/bin/java" 0
		sudo update-alternatives --install "/usr/bin/javac" "javac" "/bigdata/jdk1.8.0_191/bin/javac" 0
		sudo update-alternatives --set java /bigdata/jdk1.8.0_191/bin/java
		sudo update-alternatives --set javac /bigdata/jdk1.8.0_191/bin/javac
		java -version  ## To check
			
### Installing Hadoop					     
	
	## Download Hadoop archive file "hadoop-3.1.1.tar.gz", and follow installation steps:
		cd /bigdata
		
	## Extract the archive "hadoop-3.1.1.tar.gz", 
		tar -zxvf hadoop-3.1.1.tar.gz -C /bigdata
		
	## Setup Environment variables 
		cd   --to move to your home directory
		vi .bash_profile  --check the bash_profile file for variables
					
			Add the following under the JAVA Environment Variables section into the .bash_profile file
			---------------------------------------
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
			export PATH=$PATH:$HOME/.local/bin:$HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
			---------------------------------------

		--after save the bash_profile load it
		source .bash_profile
			
	## Create directore for Hadoop Data for (NameNode & DataNode)
		mkdir /bigdata/HadoopData
		mkdir /bigdata/HadoopData/namenode
		mkdir /bigdata/HadoopData/datanode
	
	## Configure Hadoop
		cd $HADOOP_CONF_DIR 		#check the environment variables you just added
	
	## Modify file: core-site.xml
		vi core-site.xml  --copy core-site.xml file
		
		<configuration>
		   <property>
			   <name>fs.defaultFS</name>
			   <value>hdfs://master-node:9000</value>
		   </property>
		</configuration>
		
	## Modify file: hdfs-site.xml  ## on the NameNode
		## on the NameNode server if you need DataNode, Set the parameter "dfs.datanode.data.dir"
		vi hdfs-site.xml  --copy hdfs-site.xml file
		
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
	
	## Modify file: hdfs-site.xml  
		vi hdfs-site.xml  --copy hdfs-site.xml file
			## on the other DataNode servers you have to remove "dfs.namenode.name.dir"
	
	## Modify file: mapred-site.xml  
		vi mapred-site.xml  --copy mapred-site.xml file
		
		<configuration>
		   <property>
			   <name>mapreduce.framework.name</name>
			   <value>yarn</value>
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
	
	## Modify file: yarn-site.xml  
		vi yarn-site.xml  --copy yarn-site.xml file
		
		<configuration>
		   <property>
			   <name>yarn.log.aggregation-enable</name>
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
			   <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,
			   HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
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
		</configuration>
		
	## Modify file: hadoop-env.sh       
		Edit hadoop environment file by adding the follwing environment variables under the section 
		"Set Hadoop-specific environment variables here.":  
		vi hadoop-env.sh  --copy hadoop-env.sh  
			export JAVA_HOME=/bigdata/jdk1.8.0_191
			export HADOOP_LOG_DIR=/var/log/hadoop
			export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=/bigdata/hadoop-3.1.1/lib/native"
			export HADOOP_COMMON_LIB_NATIVE_DIR=/bigdata/hadoop-3.1.1/lib/native
		
	## Create workers file
		vi workers  --copy worhers file
			## write line for each DataNode Server
			master-node 
	
	## Format the NameNode
		hdfs namenode -format
	
	## Start & Stop Hadoop
		##Start
			start-all.sh
	
		##Check hadoop processes are running
			jps 	--this command should display something like
			5968 SecondaryNameNode
			5794 DataNode
			6227 ResourceManager
			5684 NameNode
			6342 NodeManager
			6697 Jps
			
		## Default Web Interfaces
			NameNode	http://master-node:9870/ 	Default HTTP port is 9870.
			ResourceManager	http://master-node:8080/	Default HTTP port is 8080.
		
		##Stop
			stop-all.sh
>

		##################################################################################
		## 	Install Hadoop with NameNode & DataNodes on Multi Nodes	          	##
		##################################################################################					


## 1- Clone the vm created above
### Commands with root	
> login as root user

	## Turnoff firewall
		service firewalld status
		service firewalld stop
		systemctl disable firewalld
	
	## Change hostname and setup FQDN (considering the new hostname as "slave-node-1")
		# Display the hostname
		cat /etc/hostname
		
		# Edit the hostname
		vi /etc/hostname   --remove the existing file and write the below
			slave-node-1
			
		vi /etc/hosts   --your file should look like the below
			127.0.0.1		localhost	
			192.168.1.5		master-node
			192.168.1.6		slave-node-1
			
		# Type the following
		hostname slave-node-1
		
		# To check type
		hostname
			should return -> slave-node-1
		hostname -f
			should return -> slave-node-1

	## Generate SSH keys and setup password less SSH between Hadoop services
		log with hdpuser
		sudo ssh-keygen -t rsa  ## just press Enter for all choices
		cat ~/.ssh/id_rsa.pub >> ~/.ssh/id_rsa.pub/authorized_keys
		ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@master-node (you should be able to ssh without asking for password)
		ssh-copy-id -i ~/.ssh/id_rsa.pub hdpuser@xxxxxxxx (if you have more than one node, you will repeat for each node)
		ssh hdpuser@xxxx 
			- yes
		logout or exit

	## Edit the hosts file of the "master-node" vm			
		vi /etc/hosts   --your file should look like the below
			127.0.0.1	localhost	
			192.168.1.5	master-node
			192.168.1.6	slave-node-1

### Configure Hadoop				   

	## Edit the workers file into the master-node server
		vi workers  --write line for each DataNode server (in our case both server machines are considered DataNodes)
			master-node  #(if you don't want this node to be DataNode, remove this line from the workers file)
			slave-node-1

	## Modify file: hdfs-site.xml  
		If you need the data to be replicated in more than one DataNode, you must modify the replication number mentioned in the hdfs-site.xml files of all the nodes. This number cannot be greater than the number of nodes.
		vi hdfs-site.xml  --copy hdfs-site.xml file
		
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
		
	## Let's clean up some old files on both machines
		rm -rf /bigdata/HadoopData/namenode/*
		rm -rf /bigdata/HadoopData/datanode/*


## 2- Starting Hadoop					  

	## Format the NameNode on master-node
		hdfs namenode -format
	
	## Start Hadoop on master-node
		##Start
			start-all.sh

		##Check hadoop processes are running on master-node
			jps 	--this command should display something like 
			4962 NodeManager
			4851 ResourceManager
			4292 NameNode
			4407 DataNode
			5320 Jps
			4590 SecondaryNameNode
		
		##Check hadoop processes are running on slave-node-1
			jps 	--this command should display something like 
			3056 Jps
			2925 NodeManager
			2815 DataNode

	## Default Web Interfaces
		NameNode	http://master-node:9870/ 	Default HTTP port is 9870.
		ResourceManager	http://master-node:8080/	Default HTTP port is 8080.

	## Get report
		hdfs dfsadmin -report 	--this command should return something like
		``
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

		Name: 192.xxx.x.xx:9866 (master-node)
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


		Name: 192.xxx.x.xx:9866 (slave-node-1)
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
		``

	## Stop Hadoop on master-node
		stop-all.sh
