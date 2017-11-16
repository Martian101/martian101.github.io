Paxos是用来保证分布式系统一致性的协议，它将分布式系统中参与一致性决策的实例从逻辑上划分成两类角色，Proposer和Acceptor。Proposer发起提议，每个提议由提议号proposal_id和具体内容组成;Acceptor接受提议，提议通过条件是Acceptor集群超过半数的实例接受了某个提议。工程实现上一般一个实例同时承担Proposer和Acceptor两个角色。

Basic Paxos协议简单确定一个值，如将k的值设为某个值v，当某个Proposer发起的提议通过(Acceptor集群超过半数实例接受了将k设置为v的提议)后，k的值就确定了，并且协议保证这个值永远不会被更改。

多数派确定一个值，为什么需要Prepare和Accept两个阶段，一个Accept阶段达到多数派不也能确定么？我在学习Paxos的时候在这里纠结了一段时间，现在假设只有一个Accept阶段，5个实例想要确定一个值，可能的场景如下:
1 . **S1提议设置k的值为X，半数Acceptor接受后后S5发起请求设置k的值为Y**

![enter image description here](http://oojr8w6at.bkt.clouddn.com/image/png/paxos_1.png)

显然协议可以达到要求，尽管S4和S5最终将k值设置成了Y，根据法定集合性质，超过半数的Acceptor认可X值后，不会再有超过半数的Acceptor认可Y。

2 . **S1提议设置k的值为X，S5提议设置k的值为Y，S3提议设置k值为Z**

![enter image description here](http://oojr8w6at.bkt.clouddn.com/image/png/paxos_2.png)

可以看到因为没有一个请求获得多数派支持，并且以后的请求都会被拒绝，这种情况下无法通过一个值。

上述的一个Accept阶段确定一个值的过程，存在永远都无法达成一致的情况。因为Acceptor接收遇到的第一个值，当S1~S5都发起提议时，有大概率出现每个Acceptor接收的值都不一样的情况。这种情况有点类似死锁，每个进程都希望自己的提议能获取大多数Acceptor的认可却永远无法满足条件!

为了打破死锁，就需要采用某种抢占策略。现在提议中只有一个值，现实中根据提议值来作为抢占的依据不现实，意味着提议中肯定要携带额外的信息，这就是提议号。还是采用一轮协议，提议中加入提议号可以达成一致么？
为了保证全局唯一，提议号采用(自增ID + ServerId)的形式，Acceptor发现有冲突时，选择提议号更大的提议。

3 . **请求方式和场景2的时序一样，采用提议号打破死锁，最终值被确定为Y**

![enter image description here](http://oojr8w6at.bkt.clouddn.com/paxos_3.png)

但是在Y被确定的过程中，11-X和13-Z产生冲突，Acceptor接受提议号更大的13-Z，Z形成多数派被确定。也就是说，已经确定的值Z被提议号更大的Y值取代了，这样显然不符合已经确定的值不能再被更改的要求。
为了避免Acceptor反复通过不同值，就需要由Proposer提议时能够发现已经被接受的值，法定集合性质可以做到这一点，但是Proposer发现的值可能被多数派通过，也可能没有被多数派通过。如场景2: S1得到S1, S2, S3多数派回应，发现S3的提议已经被S3接受，但事实上Z的值并没有被确定。
Proposer发现已经有值被接受，就重新提议新的值为已经接受的值。如果Proposer发现Acceptor接受了两个不同的值，就选择提议号较大的值。
但是采用这种策略仍然无法打破死锁，如下面的场景4。

4 . **S1提议设置k为X, S3提议设置k为Z，S5提议设置k为Y**

![enter image description here](http://oojr8w6at.bkt.clouddn.com/paxos_4.png)

出现了S1、S2接受X, S3, S4接受Z, S5接受Y的情况，之所以会这样是因为S5在检查是否有值被接受和提议是同一个过程，检查同时挤占了Acceptor资源，最终导致再也无法形成多数派。

这个时候自然想到采用两阶段的协议，第一个阶段用来发现目前已经被Acceptor接受的协议但是并不真正提交，第二个阶段才真正发起提交。
**Prepare阶段**，得到半数的Acceptor回应后检查是否有已经被接受的值，如果有选择提议号较大的提议的值作为Accept阶段将要提交的值，否则提交自己的值。
**Accept阶段**，真正提交阶段。

![enter image description here](http://oojr8w6at.bkt.clouddn.com/image/png/paxos_5.png)

可以看到采用两阶段的协议，Prepare阶段和Accept阶段都有可能发生冲突。冲突怎么解决呢？提议号是我们拒绝的依据，Acceptor总是会接受提议号较大的提议。每个阶段都有两种冲突，Prepare阶段Acceptor接收到的提议的提议号n如果比目前接收到的提议编号都大，就承诺不再接收任何比n小的提议；如果Acceptor已经接受过Accept请求的提议，同时返回它接受过的提议号最大的Accept阶段的提议。
Accept阶段，如果Acceptor还没有接收过Accept请求的提议，就接受该请求; 如果已经接受过Accept的请求，只接受比接收过所有Accept请求的提议号都要大的提议。两个阶段都要求得到半数以上的Acceptor通过才算成功，否则Prepare阶段无法进入下一阶段即发起Accept，或者Accept失败重启该过程。
以上就是Paxos解决冲突的方式。

中文可能不太准确，Lamport大神在他的《Paxos Made Simple》原论文中对Paxos协议的表述如下:

	Phase 1
	
	(a) A proposer selects a proposal number n and sends a prepare request with number n to a majority of acceptors.
	
	(b) If an acceptor receives a prepare request with number n greater than that of any prepare request to which it has already responded, then it responds to the request with a promise not to accept any more proposals numbered less than n and with the highest-numbered pro-posal (if any) that it has accepted.
	
	Phase 2
	
	(a) If the proposer receives a response to its prepare requests (numbered n) from a majority of acceptors, then it sends an accept request to each of those acceptors for a proposal numbered n with a value v , where v is the value of the highest-numbered proposal among the responses, or is any value if the responses reported no proposals.
	
	(b) If an acceptor receives an accept request for a proposal numbered n, it accepts the proposal unless it has already responded to a prepare request having a number greater than n.
	
一个完整的Paxos流程如下图所示:
![enter image description here](http://oojr8w6at.bkt.clouddn.com/image/png/paxos.png)

总结下理解Paxos协议的几个关键点:
1. 法定集合性质，包括Prepare阶段和Accept阶段都有体现
2. 使用提议号充当"抢占式锁"的角色
3. 每个Acceptor Accept的提议号总是目前接收到的最大的提议号

因为提议号充当抢占式锁的角色，所以Paxos存在活锁的问题，可能永远无法达成一致。但不像死锁，一旦死锁就永远不可能达成一致了。

参考:

https://www.zhihu.com/question/19787937

https://youtu.be/JEpsBg0AO6o

http://codemacro.com/2014/10/15/explain-poxos/
