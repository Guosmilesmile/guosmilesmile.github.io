---
title: Redis 基础介绍
date: 2019-03-17 11:53:40
tags:
---


# Redis

http://redisdoc.com/index.html

## 什么是redis

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。     
为了实现其卓越的性能，Redis 采用运行在**内存中**的数据集工作方式。可以每隔一定时间将 数据集导出到磁盘 ， 或者 追加到命令日志中. 您也可以关闭持久化功能，将Redis作为一个高效的网络的缓存数据功能使用   


<!--more-->
## 数据类型

 - 字符串（strings）
 - 散列（hashes）
 - 列表（lists）
 - 集合（sets）
 - 有序集合（sorted sets）

## 内存管理
当某些缓存被删除后Redis并不是总是立即将内存归还给操作系统。这并不是redis所特有的，而是函数malloc()的特性。

* Reason：因为redis使用的底层内存分配器不会这么简单的就把内存归还给操作系统，可能是因为已经删除的key和没有删除的key在同一个页面（page）,这样就不能把完整的一页归还给操作系统. 
* * 当然内存分配器是智能的，可以复用用户已经释放的内存。

## Redis如何淘汰过期的keys

Redis keys过期有两种方式：被动和主动方式。

- 主动 ：当一些客户端尝试访问它时，key会被发现并主动的过期（针对单个key）。
- 被动 ： Redis每秒10次做的事情
- - 测试随机的20个keys进行相关过期检测。
- - 如果有多于25%的keys过期，重复步奏1.


## LRU（缓存淘汰算法）

##### 简称： 尝试回收最少使用的键（LRU）

触发条件：当maxmemory限制达到的时候Redis会使用的行为由 Redis的maxmemory-policy配置指令来进行配置。
* noeviction:返回错误当内存限制达到并且客户端尝试执行会让更多内存被使用的命令（大部分的写入指令，但DEL和几个例外）
* allkeys-lru: 尝试回收最少使用的键（LRU），使得新添加的数据有空间存放。
* volatile-lru: 尝试回收最少使用的键（LRU），但仅限于在过期集合的键,使得新添加的数据有空间存放。
* allkeys-random: 回收随机的键使得新添加的数据有空间存放。
* volatile-random: 回收随机的键使得新添加的数据有空间存放，但仅限于在过期集合的键。
* volatile-ttl: 回收在过期集合的键，并且优先回收存活时间（TTL）较短的键,使得新添加的数据有空间存放

##### 如何选择：

* 使用allkeys-lru策略：你希望部分的子集元素将比其它其它元素被访问的更多。如果你不确定选择什么，这是个很好的选择。.
* 使用allkeys-random：如果你是循环访问，所有的键被连续的扫描，或者你希望请求分布正常（所有元素被访问的概率都差不多）。
* 使用volatile-ttl：如果你想要通过创建缓存对象时设置TTL值，来决定哪些对象应该被过期。

##### 近似非完全：
Redis的LRU算法并非完整的实现。相反它会尝试运行一个近似LRU的算法，通过对少量keys进行取样，然后回收其中一个最好的key（被访问时间较早的）。



## Partition（分区）

##### 概念：分区是将你的数据分发到不同redis实例上的一个过程，每个redis实例只是你所有key的一个子集。


##### 目的：

* 分区可以让Redis管理更大的内存，Redis将可以使用所有机器的内存。如果没有分区，你最多只能使用一台机器的内存。
* 分区使Redis的计算能力通过简单地增加计算机得到成倍提升,Redis的网络带宽也会随着计算机和网卡的增加而成倍增长。

##### 不同的分区实现方案：

* 客户端分区就是在客户端就已经决定数据会被存储到哪个redis节点或者从哪个redis节点读取。大多数客户端已经实现了客户端分区。
* 代理分区 意味着客户端将请求发送给代理，然后代理决定去哪个节点写数据或者读数据。代理根据分区规则决定请求哪些Redis实例，然后根据Redis的响应结果返回给客户端。redis和memcached的一种代理实现就是Twemproxy
* 查询路由(Query routing) 的意思是客户端随机地请求任意一个redis实例，然后由Redis将请求转发给正确的Redis节点。Redis Cluster实现了一种混合形式的查询路由，但并不是直接将请求从一个redis节点转发到另一个redis节点，而是在客户端的帮助下直接redirected到正确的redis节点。


##### 缺点：

* 涉及多个key的操作通常不会被支持。
* 同时操作多个key,则不能使用Redis事务.
* 分区使用的粒度是key，不能使用一个非常长的排序key存储一个数据集（自我理解是因为有多个实例，如果一个非常长的排序key，切割存储就没法排序）
* 分区时动态扩容或缩容可能非常复杂。（如果Redis被当做一个持久化存储使用，必须使用固定的keys-to-nodes映射关系，节点的数量一旦确定不能变化）

## 持久化

* RDB：持久化方式能够在指定的时间间隔能对你的数据进行快照存储.（对整个内存进行dump，所以redis配置的最大内存只能是物理内存的一半）
* AOF：持久化方式记录每次对服务器写的操作,当服务器重启的时候会重新执行这些命令来恢复原始的数据。（等价trace log）
* 可以同时开启两个持久方式，优先以AOF为主。

#### RDB 优点
* RDB是一个紧凑的单一文件
* 保存RDB文件时父进程唯一需要做的就是fork出一个子进程
* 与AOF相比,在恢复大的数据集的时候，RDB方式会更快一些

#### RDB 缺点
* Redis要完整的保存整个数据集是一个比较繁重的工作,你通常会每隔5分钟或者更久做一次完整的保存,万一在Redis意外宕机,你可能会丢失几分钟的数据.
* RDB 需要经常fork子进程来保存数据集到硬盘上,当数据集比较大的时候,fork的过程是非常耗时的,可能会导致Redis在一些毫秒级内不能响应客户端的请求

https://my.oschina.net/andylucc/blog/686892
#### AOF 优点
* 可以使用不同的fsync策略：无fsync,每秒fsync,每次写的时候fsync.一旦出现故障，你最多丢失1秒的数据.
* Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写
* AOF 文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂， 对文件进行分析（parse）也很轻松[举个例子， 如果你不小心执行了 FLUSHALL 命令， 但只要 AOF 文件未被重写， 那么只要停止服务器， 移除 AOF 文件末尾的 FLUSHALL 命令， 并重启 Redis ， 就可以将数据集恢复到 FLUSHALL 执行之前的状态。]

#### fsync
有三个选项：

* 从不 fsync ：每次有新命令追加到 AOF 文件时就执行一次 fsync ：非常慢，也非常安全。
* 每秒 fsync 一次：足够快（和使用 RDB 持久化差不多），并且在故障时只会丢失 1 秒钟的数据。
* 从不 fsync ：将数据交给操作系统来处理。更快，也更不安全的选择


#### AOF 缺点
* AOF 文件的体积通常要大于 RDB 文件的体积
* 根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB。在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间
* AOF发生过bug，就是通过AOF记录的日志，进行数据恢复的时候，没有恢复一模一样的数据出来。所以说，类似AOF这种较为复杂的基于命令日志/merge/回放的方式，比基于RDB每次持久化一份完整的数据快照文件的方式，更加脆弱一些，容易有bug。

#### 快照
在默认情况下， Redis 将数据库快照保存在名字为 dump.rdb的二进制文件中。


可以通过手动执行SAVE和BGSAVE命令来执行保存快照到磁盘，SAVE和BGSAVE两个命令都会调用rdbSave函数,但它们调用的方式各有不同：
SAVE 直接调用rdbSave，阻塞Redis主进程，直到保存完成为止。在主进程阻塞期间，服务器不能处理客户端的任何请求。
BGSAVE 则fork 出一个子进程，子进程负责调用rdbSave ，并在保存完成之后向主进程发送信号，通知保存已完成。因为rdbSave 在子进程被调用，所以Redis 服务器在BGSAVE 执行期间仍然可以继续处理客户端的请求。

#### AOF 原理
AOF 重写和 RDB 创建快照一样，都巧妙地利用了写时复制机制:

* Redis 执行 fork() ，现在同时拥有父进程和子进程。
* 子进程开始将新 AOF 文件的内容写入到临时文件。
* 对于所有新执行的写入命令，父进程一边将它们累积到一个内存缓存中，一边将这些改动追加到现有 AOF 文件的末尾,这样样即使在重写的中途发生停机，现有的 AOF 文件也还是安全的。
* 当子进程完成重写工作时，它给父进程发送一个信号，父进程在接收到信号之后，将内存缓存中的所有数据追加到新 AOF 文件的末尾。
搞定！现在 Redis 原子地用新文件替换旧文件，之后所有命令都会直接追加到新 AOF 文件的末尾。

    
## 复制（主从同步）

- 异步同步
- redis的主从同步是通过增量的形式进行同步，如果同步失败，就采用全量同步的方法同步。
- 一个 master 可以拥有多个 slave、
- slave 可以接受其他 slave 的连接。除了多个 slave 可以连接到同一个 master 之外， slave 之间也可以像层叠状的结构（树状）
- Redis 复制在 master 侧是非阻塞的。
- 复制在 slave 侧大部分也是非阻塞的。加载新数据集的操作依然需要在主线程中进行并且会阻塞 slave 。
- 复制既可以被用在可伸缩性，以便只读查询可以有多个 slave 进行（读写分离 ），或者仅用于数据安全。
- 可以使用复制来避免 master 将全部数据集写入磁盘造成的开销：一种典型的技术是配置你的 master Redis.conf 以避免对磁盘进行持久化，然后连接一个 slave ，其配置为不定期保存或是启用 AOF。但是，这个设置必须小心处理，因为重新启动的 master 程序将从一个空数据集开始：如果一个 slave 试图与它同步，那么这个 slave 也会被清空。

##### 复制原理

每一个 Redis master 都有一个 replication ID ，标记了一个给定的数据集。每个 master 也持有一个偏移量，master 将自己产生的复制流发送给slave时，发送多少个字节的数据，自身的偏移量就会增加多少，目的是当有新的操作修改自己的数据集时，它可以以此更新 slave 的状态。复制偏移量即使在没有一个 slave 连接到 master 时，也会自增，所以基本上每一对给定的

Replication ID, offset

都会标识一个 master 数据集的确切版本。当 slave 连接到 master 时，它们使用 PSYNC 命令来发送它们记录的旧的 master replication ID 和它们至今为止处理的偏移量。通过这种方式， master 能够仅发送 slave 所需的增量部分。

#####replication id
每一个 Redis master 都有一个 replication ID ：这是一个较大的伪随机字符串，标记了一个给定的数据集。

##### 失败处理
但是如果 master 的缓冲区中没有足够的命令积压缓冲记录，或者如果 slave 引用了不再知道的历史记录（replication ID），则会转而进行一个全量重同步：在这种情况下， slave 会得到一个完整的数据集副本，从头开始。

##### 全量同步
master 开启一个后台保存进程，以便于生产一个 RDB 文件。同时它开始缓冲所有从客户端接收到的新的写入命令。当后台保存完成时， master 将数据集文件传输给 slave， slave将之保存在磁盘上，然后加载文件到内存。再然后 master 会发送所有缓冲的命令发给 slave。这个过程以指令流的形式完成并且和 Redis 协议本身的格式相同。
#####  主从的缺点
a)主从复制，若主节点出现问题，则不能提供服务，需要人工修改配置将从变主      
b)主从复制主节点的写能力单机，能力有限     
c)单机节点的存储能力也有限



## sentinel 哨兵(HA)

##### 为什么要有哨兵机制？
哨兵机制的出现是为了解决主从复制的缺点的


##### 主观下线和客观下线
* 主观下线（Subjectively Down， 简称 SDOWN）指的是单个 Sentinel 实例对服务器做出的下线判断。
* 客观下线（Objectively Down， 简称 ODOWN）指的是多个 Sentinel 实例在对同一个服务器做出 SDOWN 判断， 并且通过 SENTINEL is-master-down-by-addr 命令互相交流之后， 得出的服务器下线判断。

##### 每个 Sentinel 都需要定期执行的任务
1. 每隔1秒每个哨兵会向主节点、从节点及其余哨兵节点发送一次ping命令做一次心跳检测，这个也是哨兵用来判断节点是否正常的重要依据。   
![image](https://images2017.cnblogs.com/blog/1227483/201802/1227483-20180201115230609-828804435.png)
2. 每个哨兵节点每10秒会向主节点和从节点发送info命令获取最拓扑结构图，哨兵配置时只要配置对主节点的监控即可，通过向主节点发送info，获取从节点的信息，并当有新的从节点加入时可以马上感知到。
![image](https://images2017.cnblogs.com/blog/1227483/201802/1227483-20180201113518031-1426392070.png)
3. 每个 Sentinel 会以每两秒一次的频率， 通过发布与订阅功能， 向被它监视的所有主服务器和从服务器的 sentinel:hello 频道发送一条信息， 信息中包含了 Sentinel 的 IP 地址、端口号和运行 ID （runid）。同时每个哨兵节点也会订阅该频道，来了解其它哨兵节点的信息及对主节点的判断。
![image](https://images2017.cnblogs.com/blog/1227483/201802/1227483-20180201113623921-1930278582.png)
 

##### 故障转移

* 由Sentinel节点定期监控发现主节点是否出现了故障
*  * sentinel会向master发送心跳PING来确认master是否存活，如果master在“一定时间范围”内不回应PONG 或者是回复了一个错误消息，那么这个sentinel会主观地(单方面地)认为这个master已经不可用了     
![image](https://images2017.cnblogs.com/blog/1227483/201802/1227483-20180201121656031-220411144.png)
* 当主节点出现故障，此时3个Sentinel节点共同选举了Sentinel3节点为领导，负载处理主节点的故障转移
* 由Sentinel3领导者节点执行故障转移，过程和主从复制一样，但是自动执行
* * 向被选中的从服务器发送 SLAVEOF NO ONE 命令，让它转变为主服务器
* * 通过发布与订阅功能， 将更新后的配置传播给所有其他 Sentinel ， 其他 Sentinel 对它们自己的配置进行更新
* * 向已下线主服务器的从服务器发送 SLAVEOF 命令， 让它们去复制新的主服务器。
* * 通知客户端主节点已更换
* *  将原主节点（oldMaster）变成从节点，指向新的主节点


每当一个 Redis 实例被重新配置（reconfigured） —— 无论是被设置成主服务器、从服务器、又或者被设置成其他主服务器的从服务器 —— Sentinel 都会向被重新配置的实例发送一个 CONFIG REWRITE 命令， 从而确保这些配置会持久化在硬盘里。


https://www.objectrocket.com/blog/how-to/introduction-to-redis-sentinel/
Sentinels handle the failover by re-writing config files of the Redis instances that are running. Let’s go through a scenario:

Say we have a master “A” replicating to slaves “B” and “C”. We have three Sentinels (s1, s2, s3) running on our application servers, which write to Redis. At this point “A”, our current master, goes offline. Our sentinels all see “A” as offline, and send SDOWN messages to each other. Then they all agree that “A” is down, so “A” is set to be in ODOWN status. From here, an election happens to see who is most ahead, and in this case “B” is chosen as the new master.

The config file for “B” is set so that it is no longer the slave of anyone. Meanwhile, the config file for “C” is rewritten so that it is no longer the slave of “A” but rather “B.” From here, everything continues on as normal. Should “A” come back online, the Sentinels will recognize this, and rewrite the configuration file for “A” to be the slave of “B,” since “B” is the current master.
##### redis master 选举
在经历了淘汰之后剩下来的从服务器中， 我们选出复制偏移量（replication offset）最大的那个从服务器作为新的主服务器； 如果复制偏移量不可用， 或者从服务器的复制偏移量相同， 那么带有最小运行 ID 的那个从服务器成为新的主服务器。

##### 运行ID (run id)
runid是redis启动时候生成的一个随机数。

runid
 Redis "Run ID", a SHA1-sized random number that identifies a
 given execution of Redis, so that if you are talking with an instance
  having run_id = A, and you reconnect and it has run_id = B, you can be
 sure that it is either a different instance or it was restarte
 
##### sentinel leader选举
- 每个在线的哨兵节点都可以成为领导者，当它确认（比如哨兵3）主节点下线时，会向其它哨兵发is-master-down-by-addr命令，征求判断并要求将自己设置为领导者，由领导者处理故障转移；
- 当其它哨兵收到此命令时，可以同意或者拒绝它成为领导者；
- 如果哨兵3发现自己在选举的票数大于等于num(sentinels)/2+1时，将成为领导者，如果没有超过，继续选举

### salveof 
单纯的 slaveof 只是动态修改状态，不修改配置文件。

假设有两个redis，9852和9853，以及一个哨兵。
存在三个配置文件 reids9852.conf  redis9853.conf  sentinel.conf
此时9852为master，9853为slave，那么在9853的配置文件的最后，有一行slave of 9852。
sentinel在配置中会监控master，监控时会通过info信息获取slave信息，写入sentinel的配置文件中。
如果此时9852宕机，那么9853就会被选为master，此时sentinel会修改9853的配置文件，去掉slave of 这行成为master，修改sentinel的配置文件中的master和slave信息 。当9852启动的时候，因为9852在已知的slave列表中，一旦活过来，会被执行slaveof of 9853成为slave

每当一个 Redis 实例被重新配置（reconfigured） —— 无论是被设置成主服务器、从服务器、又或者被设置成其他主服务器的从服务器 —— Sentinel 都会向被重新配置的实例发送一个 CONFIG REWRITE 命令， 从而确保这些配置会持久化在硬盘里。

## redis集群

数据是否会返回经过中间那台

### MOVED 转向
一个 Redis 客户端可以向集群中的任意节点（包括从节点）发送命令请求。 节点会对命令请求进行分析， 如果该命令是集群可以执行的命令， 那么节点会查找这个命令所要处理的键所在的槽。
* 如果要查找的哈希槽正好就由接收到命令的节点负责处理， 那么节点就直接执行这个命令。
* 如果所查找的槽不是由该节点处理的话， 节点将查看自身内部所保存的哈希槽到节点 ID 的映射记录， 并向客户端回复一个 MOVED 错误。错误信息包含键 x 所属的哈希槽 3999 ， 以及负责处理这个槽的节点的 IP 和端口号 127.0.0.1:6381 。 客户端需要根据这个 IP 和端口号， 向所属的节点重新发送一次 GET 命令请求。

### HA
使用主从同步模型，为每个redis节点，配置一个或者多个salve节点。

### 缺点
* Redis集群并不支持处理多个keys的命令,因为这需要在不同的节点间移动数据,从而达不到像Redis那样的性能,在高负载的情况下可能会导致不可预料的错误.
*  集群中的节点只能使用0号数据库，如果执行SELECT切换数据库会提示错误。

### 无法保证强一致性

### Redis集群数据分片
Redis集群不同一致性哈希，它用一种不同的分片形式，在这种形式中，每个key都是一个概念性（hash slot）的一部分。
* Redis集群中有16384个hash slots，为了计算给定的key应该在哪个hash slot上，我们简单地用这个key的CRC16值来对16384取模

Redis集群中的每个节点负责一部分hash slots，假设你的集群有3个节点，那么：

* Node A contains hash slots from 0 to 5500
* Node B contains hash slots from 5501 to 11000
* Node C contains hash slots from 11001 to 16383    
允许添加和删除集群节点。比如，如果你想增加一个新的节点D，那么久需要从A、B、C节点上删除一些hash slot给到D。同样地，如果你想从集群中删除节点A，那么会将A上面的hash slots移动到B和C，当节点A上是空的时候就可以将其从集群中完全删除。

因为将hash slots从一个节点移动到另一个节点并不需要停止其它的操作，添加、删除节点以及更改节点所维护的hash slots的百分比都不需要任何停机时间。也就是说，移动hash slots是并行的，移动hash slots不会影响其它操作。



自动分配，可以指定节点上slot的数量，不能指定范围。

## 对比memcached

* 存储方式：
* * memecache 把数据全部存在内存之中，断电后会挂掉，数据不能超过内存大小 。redis有持久化模式，可以通过aof恢复数据。
* 数据支持类型：
* * redis在数据支持上要比memecache多的多
* 使用底层模型不同： 
* * 新版本的redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求。


##### 总结
有持久化需求或者对数据结构和处理有高级要求的应用，选择redis，其他简单的key/value存储，选择memcache

## redis为什么这么快


1. 完全基于内存，绝大部分请求是纯粹的内存操作，非常快速。数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；
2. 数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的
3. 采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗；
4. 使用多路I/O复用模型，非阻塞IO；
5. 使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；





