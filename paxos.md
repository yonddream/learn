#### Paxos的理解困境  
* Paxos究竟在解决什么问题？
* Paxos如何在分布式存储系统中应用？
* Paxos算法的核心思想是什么？  
 * 第一阶段做什么？  
 * 第二阶段做什么？  
 
#### Paxos和分布式存储系统
* Paxos用来确定一个不可变变量的取值。  
  * 取值可以是任意二进制数据  
  * 一旦确定将不再更改，并且可以被获取到（不可变性，可读取性）

* 在分布式存储系统中应用paxos  
  * 数据本身可变，采用多副本进行存储。  
  * 多个副本的更新操作序列[Op1,Op2,...,Opn]是相同的不变的。  
  * 用Paxos一次来去定不可变变量Opi的取值（即第i个操作是什么）  
  * 每次确定完Opi后，让每隔数据副本执行Opi，依次类推。  


#### Paxos希望解决的一致性问题
* 设计一个系统，来存储名称为var的变量。
  * 系统内部由多个acceptor组成，负责存储和管理var变量。  
  * 外部有多个proposer机器任意并发调用API，向系统提交不同的var取值。
  * var的取值可以是任意二进制数据。
  * 系统对外的API接口为：propose（var， v） => <ok, f> or <error>  

* 系统需要保证var的取值满足一致性  
  * 如果var的取值没有确定则var的取值为null  
  * 一旦var的取值被确定，则不可被更改，并且可以一直获取这个值。  

* 系统需要满足容错特性  
  * 可以容忍任意proposer机器出现故障。  
  * 可以容忍少数Accpetor故障。（半数以下）  

* 暂不考虑
  * 网络分化
  * acceptor故障会丢失var信息

#### 确定一个不可变变量---难点
* 管理多个Proposer的并发执行
* 保证var变量的不可变性
* 容忍任意Proposer机器故障
* 容忍半数以下Acceptor机器故障


在一个paxos实例中，每个提案需要有不同的编号，且编号间要存在全序关系。可以用多种方法实现这一点，例如将序数和proposer的名字拼接起来。如何做到这一点不在Paxos算法讨论的范围之内。  


#### Paxos算法说明
决议的提出与批准  
通过一个决议分为两个阶段：  

1. prepare阶段：  
  * proposer选择一个提案编号n并将prepare请求发送给acceptors中的一个多数派；
  * acceptor收到prepare消息后，如果提案的编号大于它已经回复的所有prepare消息，则acceptor将自己上次接受的提案回复给proposer，并承诺不再回复小于n的提案；
2. 批准阶段：
  * 当一个proposer收到了多数acceptors对prepare的回复后，就进入批准阶段。它要向回复prepare请求的acceptors发送accept请求，包括编号n和根据P2c决定的value（如果根据P2c没有已经接受的value，那么它可以自由决定value）。
  * 在不违背自己向其他proposer的承诺的前提下，acceptor收到accept请求后即接受这个请求。


paxos和raft差别
Paxos 相比 Raft 比较复杂和难以理解。角色扮演和流程比 Raft 都要啰嗦。比如 Agreement 这个流程，在 Paxos 里边：Client 发起请求举荐 Proposer 成为 Leader，Proposer 然后向全局 Acceptors 寻求确认，Acceptors 全部同意 Proposer 后，Proposer 的 Leader 地位得已承认，Acceptors 还得再向Learners 进行全局广播来同步。而在 Raft 里边，只有 Follower/Candidate/Leader 三种角色，角色本身代表状态，角色之间进行状态转移是一件非常自由民主的事情。Raft虽然有角色之分但是是全民参与进行选举的模式；但是在Paxos里边，感觉更像议员参政模式。







 

 