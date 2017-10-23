BigData-搭建详细说明

##Java 支撑
###1.java version（8 or 8.* 【build 1.8.0_121-b13】）

###2.spark-cassandra-connector[2.0] (spark + cassandra)版本链接：
[spark-cassandra-connector](https://github.com/datastax/spark-cassandra-connector/)

--------
#####对应版本如下
|Connector |Spark|Cassandra|Cassandra Java Driver|Minimum Java Version|Supported Scala Versions|hadoop|zookeeper|kafka|
|---------|-----------|------------------|-----------------------|--------|-----|---|---|---|
|2.0|2.0, 2.1|2.1.5*, 2.2, 3.0| 3.0 | 8|2.10, 2.11|2.7.4|3.4.8|0.10.2.1|






--------
##系统支撑
###scala(2.11.11，如果不用此开发可不需要) + spark(2.1.1) + hadoop(2.7.4)+ cassandra(3.0.13)
[scala download link](https://www.scala-lang.org/download/2.11.11.html "Title")

[spark download link](http://spark.apache.org/downloads.html "Title")

[hadoop download link](http://hadoop.apache.org/releases.html "Title")

[cassandra download link](http://cassandra.apache.org/download/ "Title")


---------
##Maven 依赖
####cassandra
	<dependency>
	    <groupId>com.datastax.cassandra</groupId>
	    <artifactId>cassandra-driver-core</artifactId>
	    <version>${com.datastax.cassandra.version}</version>
	</dependency>

-------
##系统搭建
###1.cassandra install
#####说明：

> cassandra 单点（本地）和集群都一样，代码能自动进行处理；<mark>注:多节点是ip划分（1个节点可以有多个从，即seeds中都是主节点），既集群需要多个ip

>vi ./conf/cassandra.yml
>>更改endpoint_snitch: GossipingPropertyFileSnitch
>>
>>添加auto_bootstrap: false
>>
>>########开启可修改数据中心名称和机架名称
>>broadcast_rpc_address:<LocalIp>

>add vi ./conf/cassand-env.sh
>>########开启可修改数据中心名称和机架名称
>>JVM_OPTS="$JVM_OPTS -Dcassandra.ignore_dc=true -Dcassandra.ignore_rack=true"

#####配置：[官方配置说明](http://cassandra.apache.org/doc/latest/getting_started/configuring.html 'title')
	conf/cassandra.yaml（主配置文件，包括日志路径，数据存放路径，各种参数配置等）
	
	$ vim conf/cassandra.yaml
	#cassandra 的精简配置，可以运行集群的最低配置。
	cluster_name: 'My Cluster'            #集群名
	num_tokens: 256
	seed_provider:
	    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
	      parameters:
	          - seeds: "10.10.8.3,10.10.8.4,10.10.8.5"   #节点ip列表
	rpc_address: 10.10.8.5                    #远程过程调用（集群时其他机器可以访问本机）
	listen_address: 10.10.8.5                 #进程监听地址
	storage_port: 7000                        #集群中节点通信的端口号
	start_native_transport: true              #开启native协议
	native_transport_port: 9042               #客户端的交互端口
	 
	data_file_directories:
	    - /data/cassandra/dbdata              # 数据位置，多盘的话可以写多个目录
	commitlog_directory:
	    - /data/cassandra/commitlog           #commitlog的路径，与data目录分开磁盘，提高性能
	saved_caches_directory:
	    - /data/cassandra/caches              #缓存数据目录
	hints_directory:
	    - /data/cassandra/hints
	commitlog_sync: batch                     #批量记录commitlog，每隔一段时间将数据commitlog
	commitlog_sync_batch_window_in_ms: 2      #batch模式下，批量操作缓存的时间间隔
	#commitlog_sync: periodic                 #周期记录commitlog，每一次有数据更新都commitlog
	#commitlog_sync_period_in_ms: 10000       #periodic模式，刷新commitlog的时间间隔


###2.hadoop install
#####说明：
	mac os:(伪分布式操作)
	1.安装路径为：cd /usr/local/Cellar/hadoop-2.7.4
	2.【必选】需要在"系统偏好设置"里面开启"远程登录"，如果不开启会出现错误:ssh: connect to host 0.0.0.0 port 22: Connection refused
	3.【可选】./sbin/start-fds.sh时如果需要剔除警告（WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable），请在路径“./etc/hadoop/log4j.properties”最后添加以下这句
	log4j.logger.org.apache.hadoop.util.NativeCodeLoader=ERROR
	
	linux：暂时未安装
#####配置：[官方配置说明](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html 'title') , [博客配置说明](http://www.powerxing.com/install-hadoop/ 'title')

1.[Configuration进行配置](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#Configuration)
	<value>hdfs://localhost:9000</value> 为hdfs的访问路径(spark可进行访问)

2.[Setup passphraseless ssh](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#Setup_passphraseless_ssh 'title')
	【mac 不需根据链接进行】只需在"系统偏好设置"里面开启"远程登录"，如果不开启会出现错误:ssh: connect to host 0.0.0.0 port 22: Connection refused
	
3.[Execution 执行](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#Execution 'title')
	
	注意：
	$ bin/hdfs dfs -mkdir /user
  	$ bin/hdfs dfs -mkdir /user/<username>
  	上面的<username>对应计算机名称
  	
  	使用下面的命令替代上面的命令：
  	./bin/hdfs dfs -mkdir -p /user/BennieSun/input
 
4.[YARN on a Single Node](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#YARN_on_a_Single_Node 'title')

5.环境变量配置：HADOOP_HOME=$BASE_CELLAR/hadoop-2.7.4

**注意命令**

* ./bin/hdfs namenode -format ---> 启动namenode
* jps 查看是否启动namenode
* ./sbin/start-dfs.sh ---> 启动dfs
* ./sbin/start-yarn.sh
* 或者用start-all.sh启动

###2.zookeeper install [参考链接](http://www.cnblogs.com/haippy/archive/2012/07/19/2599989.html 'title')
#####说明：
	mac os:(伪集群操作)
	1.为集群使用3个端口
	2.zookeeper能做hadoop hdfs的管理
#####操作
* 复制至少3个（解压文件夹到目录）解压的文件夹
* 修改3个文件夹中的./conf/zoo.cfg（修改dir、logdir、clientPort【需要不一样的端口】），添加伺服器配置
	
	dataDir=/Users/joesun/zookeeper/server0/tmp/data
	dataLogDir=/Users/joesun/zookeeper/server0/tmp/log
	
	server.0=127.0.0.1:2287:3387
	server.1=127.0.0.1:2288:3388
	server.2=127.0.0.1:2289:3389

**注意命令**

* 启动命令
* zookeeper-3.4.8-cluster0/bin/zkServer.sh start
* zookeeper-3.4.8-cluster1/bin/zkServer.sh start
* zookeeper-3.4.8-cluster2/bin/zkServer.sh start


###2.spark install【可链接集群Hadoop HDFS】  [参考链接](http://blog.csdn.net/stark_summer/article/details/43495623 'title')
#####说明：
	mac os:(伪集群操作)
	spark链接集群master uri：spark://10.10.10.200:7077

#####配置
	mv ./conf/spark-env.sh.template ./conf/spark-env.sh
	vi ./conf/spark-env.sh

	添加以下内容：
	export SPARK_WORKER_MEMORY=1G
	export HADOOP_CONF_DIR=/usr/local/Cellar/hadoop-2.7.4/etc/hadoop
	export SPARK_MASTER_IP=127.0.0.1
	
**注意命令-启动spark伪分布式**

* 第一步 先保证Hadoop集群或者伪分布式启动成功，如果没有启动，进入hadoop的sbin目录执行 ./start-all.sh
* 进入spark的sbin目录下执行“start-all.sh”
* 【成功标识】看到有新进程“Master” 和"Worker"
* localhost:8080 能看到界面

###2.kafka install【需要集群zookeeper】  [参考链接](http://blog.csdn.net/u012373815/article/details/52752654 'title')
#####说明：
	mac os:(伪集群操作)
	default.replication.factor=2 这样的目的是当生产者链接leader，leader挂掉后，消息未备份，导致不能继续

#####配置[url](http://blog.csdn.net/w13770269691/article/details/38706747 'title')
	cp ./config/server.properties ./config/server-1.properties
	cp ./config/server.properties ./config/server-2.properties

	vi ./config/server.prop
	修改以下内容：
		log.dir=~/kafka/tmp/kafka-logs
		broker.id=0
	新增以下内容：
		port=9092
		host.name=127.0.0.1
		advertised.host.name = localhost
		#消息备份数目 默认1不做复制，建议修改
		default.replication.factor=2
	开启：
		delete.topic.enable=true

	vi ./config/server-1.prop
	修改以下内容：
		broker.id=1
		log.dir=~/kafka/tmp1/kafka-logs
	新增以下内容：	
		port=9093
		host.name=127.0.0.1
		advertised.host.name = localhost
		#消息备份数目 默认1不做复制，建议修改
		default.replication.factor=2
	开启：
		delete.topic.enable=true

	vi ./config/server-2.prop
	修改以下内容：
		broker.id=2
		log.dir=~/kafka/tmp2/kafka-logs
	新增以下内容：
		port=9094
		host.name=127.0.0.1
		advertised.host.name = localhost
		#消息备份数目 默认1不做复制，建议修改
		default.replication.factor=2
	开启：
		delete.topic.enable=true
	
	创建以上的～／kafka/tmp ~/kafka/tmp1 ~/kafak/tmp2

**注意命令-启动spark伪分布式**

* bin/kafka-server-start.sh config/server.properties
* bin/kafka-server-start.sh config/server-1.properties
* bin/kafka-server-start.sh config/server-2.properties  

**test**

>kafka启动后我门可以新建一个topic （topic的名字为testyang，–replication-factor表示复制到多少个节点，–partitions表示分区数，一般都设置为2或与节点数相等，不能大于总节点数）
>>**--创建topic**
>>>* bin/kafka-topics.sh --create --zookeeper 127.0.0.1:2181 -replication-factor 1 --partition 2 --topic testyang
>>>

>>**--查看topic**
>>>* bin/kafka-topics.sh --list --zookeeper 127.0.0.1:2181
>>>

>>**--查看topic状态**
>>>bin/kafka-topics.sh --describe --zookeeper 127.0.0.1:2181 --topic test_node2
>>>

>>**--创建一个producer 发送数据**
>>>* ./bin/kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic test_node2
>>>

>>**--创建一个consumer 消费数据**
>>>* ./bin/kafka-console-consumer.sh --zookeeper 127.0.0.1:2181 --topic test_node2 --from-beginning
>>>

>>**--kafka删除topic方法**
>>>* ./bin/kafka-topics.sh --delete --zookeeper 127.0.0.1:2181 --topic abelyang

>>**--**
>>>bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test_node2 --from-beginnin

