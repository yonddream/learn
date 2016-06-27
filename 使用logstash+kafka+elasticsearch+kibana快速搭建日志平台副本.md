### zookeeper
#### 1. 单机模式启动
$ bin/zkServer start  
ZooKeeper JMX enabled by default  
Using config: /usr/local/etc/zookeeper/zoo.cfg  
Starting zookeeper ... STARTED  
上面就启动了zookeeper，它会自动去寻找zoo.cfg配置文件来进行启动。  

#### 2. 集群模式
##### 2.1 配置zoo.cfg文件
我们看看集群模式下zoo.cfg的配置，下面是单机起3个zookeeper的其中一个配置。  
<pre><code>
tickTime=2000
initLimit=10
syncLimit=5
clientPort=2182
dataDir=/usr/local/var/run/zookeeper/data-node2
server.1=localhost:2887:3887
server.2=localhost:2888:3888
server.3=localhost:2889:3889
</code></pre>
`tickTime=2000:`  
tickTime这个时间是作为Zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔,也就是每个tickTime时间就会发送一个心跳；  

`initLimit=10:`  
initLimit这个配置项是用来配置Zookeeper接受客户端（这里所说的客户端不是用户连接Zookeeper服务器的客户端,而是Zookeeper服务器集群中连接到Leader的Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。  
当已经超过10个心跳的时间（也就是tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息,那么表明这个客户端连接失败。总的时间长度就是 10*2000=20 秒；  

`syncLimit=5:`  
syncLimit这个配置项标识Leader与Follower之间发送消息,请求和应答时间长度,最长不能超过多少个tickTime的时间长度,总的时间长度就是5*2000=10秒；   

`dataDir=/usr/local/var/run/zookeeper/data-node2`
dataDir顾名思义就是Zookeeper保存数据的目录,默认情况下Zookeeper将写数据的日志文件也保存在这个目录里；  

`clientPort=2181`  
clientPort这个端口就是客户端连接Zookeeper服务器的端口,Zookeeper会监听这个端口接受客户端的访问请求；  

`server.1=localhost:2887:3887`  
`server.2=localhost:2888:3888`  
`server.3=localhost:2889:3889`  
`server.A=B：C：D：`  
A是一个数字,表示这个是第几号服务器,B是这个服务器的ip地址  
C第一个端口用来集群成员的信息交换,表示的是这个服务器与集群中的Leader服务器交换信息的端口  
D是在leader挂掉时专门用来进行选举leader所用  

##### 2.2 创建ServerID标识
除了修改zoo.cfg配置文件,集群模式下还要配置一个文件myid,这个文件在dataDir目录下,这个文件里面就有一个数据就是A的值,在上面配置文件中zoo.cfg中配置的dataDir路径中创建myid文件。
<pre><code>
cat /usr/local/var/run/zookeeper/data-node1/myid
1
</code></pre>

##### 2.3 启动zookeeper集群
<pre><code>
cd /usr/local/Cellar/zookeeper/3.4.7
bin/zkServer start /usr/local/etc/zookeeper/zoo1.cfg 
bin/zkServer start /usr/local/etc/zookeeper/zoo2.cfg 
bin/zkServer start /usr/local/etc/zookeeper/zoo3.cfg 
</code></pre>


##### 2.4 检测集群是否启动
使用端口查看状态  
`echo stat|nc localhost 2181`  
直接连接到zookeeper  
`cd /usr/local/Cellar/zookeeper/3.4.7 && bin/zkCli`
 
### kafka
kafka依赖于zookeeper，它有自带的zookeeper但是我们上面已经装好了zookeeper的集群，所以我们使用上面的集群来进行配置。  
#### 1.单点启动kafka
kafka的配置文件中默认只有一个zookeeper是本地地址，可以修改成集群，集群中有几个就写几个。   
`zookeeper.connect=localhost:2181,localhost:2182,localhost:2183`  
启动配置  
`bin/kafka-server-start /usr/local/etc/kafka/server.properties`

#### 2.创建topic
下面是创建了一个只有一个副本，分区为1的topic "test"。    
`bin/kafka-topics --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test --create`  
查看创建的topic  
`bin/kafka-topics --zookeeper localhost:2181 --describe`

#### 3.删除topic
kafka默认设置是不能删除topic的，只会将topic标识为删除状态。修改配置可以直接删除topic。  
`delete.topic.enable = true`  
修改配置后重启kafka。删除topic  
`bin/kafka-topics --zookeeper localhost:2181 --topic test --delete`  

#### 4.收发信息
发送信息  
`bin/kafka-console-producer --broker-list localhost:9092 --topic test`
接收信息  
`bin/kafka-console-consumer --zookeeper localhost:2181 --topic test --from-beginning`  

#### 5.多broker测试
复制2份config，然后修改下配置。  
<pre><code>
config/server-1.properties:
    broker.id=1
	listeners=PLAINTEXT://:9094
    log.dirs=/usr/local/var/lib/kafka-logs-1
</code></pre>

#### 6.多节点启动
JMX_PORT是为了打开kafka的可管理端口（默认不启用）。所以需要配置不同的端口。  
<pre><code>
cd /usr/local/Cellar/kafka/0.9.0.0
JMX_PORT=9997 bin/kafka-server-start /usr/local/etc/kafka/server1.properties &
JMX_PORT=9998 bin/kafka-server-start /usr/local/etc/kafka/server2.properties &
JMX_PORT=9999 bin/kafka-server-start /usr/local/etc/kafka/server3.properties &
</code></pre>
创建一个带有多个副本的topic    

`bin/kafka-topics --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic mytopic --create`  

#### 7.查看topic的状态  
<pre><code>
bin/kafka-topics --zookeeper localhost:2181  --desc
Topic:mytopic	PartitionCount:1	ReplicationFactor:3	Configs:
	Topic: mytopic	Partition: 0	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
</code></pre>
partition：同一个topic下可以设置多个partition，将topic下的message存储到不同的partition下，目的是为了提高并行性  
leader：负责此partition的读写操作，每个broker都有可能成为某partition的leader  
replicas：副本，即此partition在那几个broker上有备份，不管broker是否存活  
isr：存活的replicas  