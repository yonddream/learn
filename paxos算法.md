##### 简介
在分布式算法领域，有个非常重要的算法叫Paxos, 它的重要性有多高呢，Google的Chubby[1]论文中提到    

>all working protocols for asynchronous consensus we have so far encountered have Paxos at their core.    

Paxos算法是由Leslie Lamport提出的，他在《Paxos Made Simple》[2]中写道
>The Paxos algorithm, when presented in plain English, is very simple.    

大神Lamport的第一篇论文《The Part-Time Parliament》[3]用在虚构的希腊岛屿Paxos上的人们通过议会表决法律来解释Paxos算法，群众纷纷表示太难理解了。大神表示你们这群渣渣不懂我的幽默，既然如此，我就用简明英语再表述一遍。大神Lamport说非常简单，但是Paxos公认的繁琐难懂，尤其是要用程序员严谨的思维将所有细节理清的时候，你的脑袋里更是会充满了问号。 

希望这篇文章能让一个和我一样的小白能理解Paxos究竟在解决什么问题？Paxos算法的核心思想是什么？Paxos的2阶段都做了什么？

##### Paxos是什么  
&ensp; &ensp; 分布式系统中的节点通信存在两种模型：共享内存（Shared memory）和消息传递（Messages passing）。基于消息传递通信模型的分布式系统，不可避免的会发生以下错误：进程可能会慢、被杀死或者重启，消息可能会延迟、丢失、重复，在基础Paxos场景中，先不考虑可能出现消息篡改即拜占庭错误的情况。Paxos算法解决的问题是在一个可能发生上述异常的分布式系统中如何就某个值达成一致，保证不论发生以上任何异常，都不会破坏决议的一致性（来源[维基百科] [wiki][4]）。   

[wiki]: https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95  (维基百科)

&ensp; &ensp; 简单点说Paxos的目的是让整个集群的结点对某个值的变更达成一致。Paxos可以说是一个民主选举的算法——大多数节点的决定会成个整个集群的统一决定。任何一个点都可以提出要修改某个数据的提案，是否通过这个提案取决于这个集群中是否有超过半数的结点同意。取值一旦确定将不再更改，并且可以被获取到（不可变性，可读取性）。  


##### Paxos各角色的职能
Proposer ：提议者，提出议案（同时存在一个或者多个，他们各自发出提案）；  
Acceptor：接受者，收到议案后选择是否接受；  
Client：产生议题者，发起新的请求；     
Learner：最终决策学习者，只学习正确的决议；   

&ensp; &ensp; 上面4种角色中最主要的是Proposer和Acceptor。Proposer就像Client的使者，由Proposer使者拿着Client的议题去向Acceptor提议，让Acceptor来决策。主要的交互过程在Proposer和Acceptor之间。  
下面用一幅图来标识角色之间的关系。  
<img src="https://raw.githubusercontent.com/yonddream/learn/master/paxos/relationship.png" width = "300" alt="角色关系" align=center />    

上图中是画了很多节点的，每个节点需要一台机器么？答案是不需要的，上面的图是逻辑图，物理中，可以将Acceptor和Proposer以及Client放在一台机器上，Acceptor启动端口进行TCP监听，Proposer来主动连接即可。所以完全可以将Client、Proposer、Acceptor、Learner合并到一个程序里面。     

#### Paxos算法内容
我们先看看Paxos在原作者的《Paxos Made Simple》中的描述。  
决议的提出与批准  
通过一个决议分为两个阶段：  

###### 1. prepare阶段  
1. proposer选择一个提案编号n并将prepare请求发送给acceptors中的一个多数派；
* acceptor收到prepare消息后，如果提案的编号大于它已经回复的所有prepare消息，则acceptor将自己上次接受的提案回复给proposer，并承诺不再回复小于n的提案；  

下图是一个proposer和5个acceptor之间的交互，对2种不同的情况做了处理。    
<img src="https://raw.githubusercontent.com/yonddream/learn/master/paxos/prepare.png" width = "400" alt="角色关系" align=center />   


###### 2. 批准阶段：
1. 当一个proposer收到了多数acceptors对prepare的回复后，就进入批准阶段。它要向回复prepare请求的acceptors发送accept请求，包括编号n和value。    
* 在不违背自己向其他proposer的承诺的前提下，acceptor收到accept请求后即接受这个请求。  

<img src="https://raw.githubusercontent.com/yonddream/learn/master/paxos/accept.png" width = "400" alt="角色关系" align=center /> 

可以看出，Proposer与Acceptor之间的交互主要有4类消息通信，这4类消息对应于paxos算法的两个阶段4个过程。用2轮RPC来确定一个值。上面的图解都只是一个Proposer，但是实际中肯定是有其他Proposer针对同一件事情发出请求，所以在每个过程中都会有些特殊情况处理，这也是为了达成一致性所做的事情。如果在整个过程中没有其他Proposer来竞争，那么这个操作的结果就是确定无异议的。但是如果有其他Proposer的话，情况就不一样了。  

#### 唯一编号
保证Paxos正确运行的一个重要因素就是提案(proposal)编号，编号之间要能比较大小/先后，如果是一个proposer很容易做到，如果是多个proposer同时提案，该如何处理？Lamport不关心这个问题，只是要求编号必须是全序的，但我们必须关心。这个问题看似简单，实际还稍微有点棘手，因为这本质上是也是一个分布式的问题。  

在Google的Chubby论文中给出了这样一种方法：  
假设有n个proposer，每个编号为ir (0 <= ir < n)，proposol编号的任何值s都应该大于它已知的最大值，并且满足：`s % n = ir => s = m * n + ir`

proposer已知的最大值来自两部分：proposer自己对编号自增后的值和接收到acceptor的reject后所得到的值。    
我们以3个proposer P1、P2、P3为例。  
开始m=0,编号分别为0，1，2。
P1提交的时候发现了P2已经提交，P2编号为1 > P1的0，因此P1重新计算编号：`new P1 = 1 \* 3 + 0 = 4`  
P3以编号2提交，发现小于P1的4，因此P3重新编号：`new P3 = 1 * 3 + 2 = 5`  
 
整个paxos算法基本上就是围绕着proposal编号在进行：proposer忙于选择更大的编号提交proposal，acceptor则比较提交的proposal的编号是否已是最大，只要编号确定了，所对应的value也就确定了。所以说，在paxos算法中没有什么比proposal的编号更重要。


##### Multi Paxos
Paxos对某一个问题达成一致的一个协议.《Paxos Made Simple》[2]花大部分的时间解释的就是这个一个提案的问题, 然后在结尾的Implementing a State Machine章节介绍了我们大部分的应用场景是对一堆连续的问题达成一致。  

最简单的方法就是实现每一个问题独立运行一个Paxos的过程（instance）。每个过程都是独立的，相互不会干扰，这样可以为一组连续的问题达成一致。但是这样每一个问题都需要Prepare, Accept两个阶段才能够完成。Prepare阶段频繁请求会造成无谓的浪费，我们能不能把这个过程给减少。  

这样就引入Proposer Leader的选举，正常的Paxos二阶段从Proposer Group中选举出Leader后，后续统一由Leader发起提案，只有Leader才能发起提案的话相当于Proposer只有一个，所以可以省略Prepare阶段直接进入到Accpet阶段。直至发生Leader宕机、重新进行选举。  

《Paxos Made Live》[5]论文中讲解了如何使用multi paxos实现chubby的过程，以及实现过程中需要解决的问题，比如需要解决磁盘冲突，如何优化读请求，引入了Epoch number等。  

##### 实际应用
从上面我们知道，Paxos在节点宕机恢复、消息无序或丢失、网络分化的场景下能保证决议的一致性。而Paxos的描述侧重于理论，在实际项目应用中，处理了N多实际细节后，可能已经变成了另外一种算法，这时候正确性已经无法得到理论的保证。要证明分布式一致性算法的正确性通常比实现算法还困难。所以很多系统实际中使用的都是以Paxos理论为基础而衍生出来的变种和简化版。例如Google的Chubby、MegaStore、Spanner等系统，ZooKeeper的ZAB协议，还有更加容易理解的raft协议。大部分系统都是靠在实践中运行很长一段时间才能谨慎的表示，系统已基本运行，没有发现大的问题。  


##### 参考文献
1. [The Chubby lock service for loosely-coupled distributed systems.  Mike Burrows, Google Inc] [Chubby]  ----  Google Chubby论文  
2. [Paxos Made Simple.  Leslie Lamport] [paxos]  ----   2001年。   
注：Lamport觉得同行无法接受他的幽默感，于是用容易接受的方法重新表述了一遍。
3. [The Part-Time Parliament. Leslie Lamport] [paxos-first]  ----  Lamport于1998年发表在ACM Transactions on Computer Systems。  
注：这是该算法第一次公开发表。
4. [Paxos算法维基百科] [wiki]  ----  里面详细的描述了Paxos的论证过程。  
5. [Paxos Made Live - An Engineering Perspective] [paxos-live]  ----  Multi Paxos实际应用  
6. [lamport-paxos] [lamport]  ----  描写了他用9年时间发表这个算法的前前后后   



[Chubby]: http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/chubby-osdi06.pdf  (Chubby论文)
[paxos]: http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf  (paxos论文)
[paxos-first]: http://research.microsoft.com/en-us/um/people/lamport/pubs/lamport-paxos.pdf  (paxos论文)
[paxos-live]: https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/paper2-1.pdf  (Multi Paxos)
[lamport]: http://research.microsoft.com/users/lamport/pubs/pubs.html#lamport-paxos (Lamport发表论文的9年趣事)