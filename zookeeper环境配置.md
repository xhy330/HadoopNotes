# zookeeper环境配置

## 1. zookeeper最初印象

zookeeper是一个头头，用来管理集群状态的。比如在hdfs集群中，如果nn1挂了，则可以通过zookeeper来选举其他人来当头头。为了让自己稳定下来，zookeeper也是集群的形式而存在的，如果某个机器宕机了，则可以通过内部选举，选出一个zookeeper的leader，继续保持对其他集群的服务。

## 2. zookeeper的集群

其中的每一个server可连接不同的集群，并且zookeeper内部之间的数据要保持一致，如果某个server坏了，则可以临时更换一个新的server继续服务。

## 3. zookeeper的角色

* **领导者(leader)：**负责发起投票和决议，更新系统状态，发送心跳之类的。
* **跟随者(follower)：**用于接受客户端请求、向客户端返回结果，在选举中参与投票，结果由leader产生。
* **观察者(Observer)：**可以接受客户端请求，将写请求转发给leader，但observer不参加投票过程，只同步leader的状态，observer的目的是为了扩展系统，提高读取速度。observer不属于法定人数，即不参加选举也不响应提议。

比如公司有200台机器，只有1个leader，如果选举的话，199个follower将会花费很多时间。不如选择少的follower，其他的换成observer进行监控。

**注意：** 

1. leader失效后只会在follower中进行选举

	         2. 每个follower都和leader连接，接受leader的数据更新
	         2. 客户端可以连接到每个server，每个server的数据是一模一样的

## 4. zookeeper的搭建

### 4.1 规划

选择nn1、nn2、s1来进行实验，一个leader和两个follower。选择三个的原因就是选举的时候要求票数要过一半，所以至少是三台机器

### 4.2 zookeeper的机器批量分布脚本

**思路：就是在原来的基础上，把5个机器变成了三台机器。**

1. 生成新的只有三台机器的ip文件
   ```shell
   cp ips ips_zk
   vim ips_zk
   # 再手动将ip进行调整规划
   ```

2. 生成新的zk脚本
   ```shell
   # 复制原来的脚本，修改内容
   cp ssh_all.sh ssh_all_zk.sh
   cp ssh_root.sh ssh_root_zk.sh
   cp scp_all.sh scp_all_zk.sh
   
   # 打开脚本，将ips修改为ips_zk
   ```

### 4.3 安装zookeeper

1. 找到安装包
   ```shell
   /public/software/bigdata/zookeeper-3.4.8.tar.gz
   ```

2. 把所有机器上的zookeeper的tar包解压到/usr/local目录下
   ```shell
   ssh_root_zk.sh tar -zxvf /public/software/bigdata/zookeeper-3.4.8.tar.gz -C /usr/local/
   ```

3. 创建软连接
   ```shell
   ssh_root_zk.sh ln -s /usr/local/zookeperp-3.4.8 /usr/local/zookeeper
   ```

4. 修改zookeeper权限，我们是要使用hadoop用户进行操作
   ```shell
   ssh_root_zk.sh chown hadoop:hadoop -R /usr/local/zookeeper-3.4.8
   ```

### 4.4 修改每个zookeeper的配置

zookeeper的脚本文件都是放在bin目录里面的，要使用还得配置环境变量才行。配置文件则都放好在了conf目录里面的zoo.cfg里

```shell
mv zoo_sample.cfg zoo.cfg
# 否则的话不会生效
```

**注意几个参数**

* **ticktime：**zk中的时间单元，zk中所有的时间都是以它为基础。
* **initLimit：**follower在从leader同步所有最新数据，然后确定自己对外服务的状态。
* **syncLimit：**Leader负责跟ZK集群中所有机器通信，例如心跳检测来确定机器是否还存活。如果超过syncLimit这个时间，则就是认为不在线了。
* **clientPort：**客户端连接server的端口，即对外服务的端口，一般设置为2181
* **dataDir：**存储事务日志文件
* **dataLogDir：**事务日志输出目录
* **server.x=[hostname]:nnnn[:nnnn] 	：**x是一个数字，跟myid中的id是保持一致的，右边配置两个端口，第一个用来Leader与Follower进行通信，另一个用来进行Leader选举的投票通信

```shell
#修改配置文件名称
mv /usr/local/zookeeper/conf/zoo_sample.cfg /usr/local/zookeeper/conf/zoo.cfg
vim /usr/local/zookeeper/conf/zoo.cfg
#增加图上内容
server.1=nn1:2888:3888
server.2=nn1:2888:3888
server.3=nn1:2888:3888
#其中1 2 3分别代表的节点编号 2888节点之间的通信端口 3888节点间的选举端口
dataDir=/data
#zookeeper运行时候的数据存储位置
```

### 4.5 修改输出日志配置文件所在目录

ZK默认是将日志输出到当前目录，所有要进行手动修改然后统一管理

将zookeeper/bin/zkEnv.sh脚本中的ZOO_LOG_DIR设置为/data

```shell
#拷贝zkEnv.sh到每台机器的zookeeper的bin目录下
scp_all_zk.sh /usr/local/zookeeper/bin/zkEnv.sh /usr/local/zookeeper/bin/ 
```

### 4.6 创建data目录，并且生产myid文件

```shell
# 5个都安装，比较后面其他的机器也会用到
ssh_root.sh mkdir /data
# 修改权限，要不然只能是root用户才能使用data
ssh_root.sh chown hadoop:hadoop -R /data
# 验证
ssh_root.sh "ls -l / | grep data"
```

生成myid文件

```shell
ssh_all_zk.sh touch /data/myid
# 分别是不同的机器
echo 1 > /data/myid
echo 2 > /data/myid
echo 3 > /data/myid
```

### 4.7 为了让ZK能够被调用，设置环境变量

```shell
#在nn1 nn2 s1的机器上面执行以下操作
echo 'export ZOOKEEPER_HOME=/usr/local/zookeeper' >> /etc/profile
echo 'export PATH=$PATH:$ZOOKEEPER_HOME/bin' >> /etc/profile
ssh_root_zk.sh source /etc/profile
# 然后打出zk后按tab，查看是否有提示
```

### 4.8 启动zookeeper

```shell
# 批量启动
ssh_all_zk.sh /usr/local/zookeeper/bin/zkServer.sh start
# 批量查看java进程则用jps
ssh_all_zk.sh jps
# 查询各个zookeeper节点的状态
ssh_all_zk.sh /usr/local/zookeeper/bin/zkServer.sh status
```

**补充：前台执行与后台执行**

nohup命令是后台一直执行，即使切换了用户也执行，&则是在后台执行，切换后就全部暂停。
linux进程中，2代表的是错误的输出，1代表的则是正确的输出。不过不想查看执行日志，可以把全部输出都放在/dev/null下面

### 4.9 查看zookeeper输出日志和进程消息

```shell
# 文件放在
/data/zookeeper.out
```

由于ZooKeeper集群启动的时候，每个结点都试图去连接集群中的其它结点，先启动的肯定连不上后面还没启动的，所以上面日志前面部分的异常是可以忽略的。通过后面部分可以看到，集群在选出一个Leader后，最后稳定了。 其他结点可能也出现类似问题，属于正常。

### 4.10 通过进程ID查看日志存放位置

有时候在陌生的环境下面不知道日志放在了哪里，所以就需要自己手动的来查看

```shell
# 使用jps查看进程id
jps
# 通过进程id查看linux服务运行日志
cd /proc/id/fd
```

## 5 zookeeper的初始化选举和数据模型

### 5.1 选举

zookeeper的选举其实是贪婪模式的选举，然后票数过半者就是Leader，以三台机器为例

1. 节点的启动顺序是server1、server2、server3
2. zookeeper的选举都是贪婪模式选举，每个人启动先给自己投票
3. Server1，也就是myid=1的机器(其他机器同理)首先启动，给自己投一票，此时自己获取一票
4. sercer2启动服务器，也给自己投一票
5. **server1和server2要交换选票。server1发现server2的myid要比自己的大，所以就把自己的票给了server2，此时server1的票数为0，server2的票数为2**
6. server2的票数过半，则选举为Leader
7. server3启动了，但是发现集群中已经有了Leader，故加入该Leader，自己成为Follower

### 5.2 数据模型

zookeeper本身是一个分布式基于内存的文件系统。
它不同于Windows系统C盘之类的里面可以存放文件，它只能存放文件夹也就是路径，不能放文件！但是可以对文件夹进行挂载数据，最大不超过1M。zookeeper本身可以理解成为一个木棒，本身没啥用，主要是看你怎么来使用它。由于是文件系统，故文件夹(ZNode)不能重名，可以看成是一个数据库！

## 6. zookeeper的操作

### 6.1 连接zookeeper

```shell
# zookeeper客户端的连接命令
# zkCli是zookeeper内正宗的客户端。-server表示启动服务，连接上集群对外提供的端口2181(这个是文件配置中设置的)
zkCli.sh -server nn1:2181,nn2:2181,s1:2181
```

提示connected表示连接成功。注意，联机这集群的不一定是非要客户端，可以是其他的集群机器。因为其文件系统的特性，我们的操作无非就是增删改查其中的值，对比数据库，以路径为key，描述信息为value。连接三台机器是为了防止某个宕机了，则客户端就连不上了。

### 6.2 zookeeper内的命令

基于文件的内存系统，文件上有节点的信息以及可以往里面设置值，最多只有1M，对文件系统做操作。

stat 查看节点状态信息 
set  设置节点的值 
ls   查看子节点
ls2  查看子节点并且查询状态信息 （stat + ls）
delete 软删除节点如果存在子节点不可以删除 
sync 同步节点数据信息
rmr  强制删除节点
get  获取节点的值 
create 创建节点
quit 退出客户端 
close 关闭客户端和服务端的链接
connect 链接服务端 

### 6.3 节点的操作

```shell
#创建节点 操作语法
create [-s序列节点] [-e临时节点] path data


# zookeeper中节点类型分为4种
#持久节点是创建完毕的节点在zookeeper中一直存在的
#持久无序列节点 
create /node1 hainiu
#出现的结果就是 /node1
#持久带顺序的节点
create -s /node1 hainiu
#出现结果 /node10000000001 会自动在节点后面增加序列号
#临时节点，在客户端连接的时候使用的时候会一直存在，但是客户端退出的时候节点会自动消失
create -e /tmp tmp
#临时带有序列的节点
create -e -s /tmp tmp
#结果是/tmp0000000251自带事物编号


#close关闭客户端和服务端的链接
close
#重新链接 connect ip:port
connect nn1:2181.nn2:2181,s1:2181
#查看根节点的节点信息 ls /
lS /
#可以看到临时节点全部都消失了


#quit可以直接退出客户端
quit
# close是关闭客户端和服务端的链接但是不会退出客户端


#set设置节点的值
set path value
#get获取节点的值
get path
```

设置值和获取值种所看到的节点中的详情信息

这里我们引申出来一个新的概念：**zxid（事物id）**

> zxid相当于是mysql等数据库中的事物，每次对于zookeeper中的数据或者是节点做更改操作的时候都会存在一个zxid进行记录其中的数据的版本，版本永远递增不会减小，在zookeeper中所有节点的数据版本都应该保持一致，所以所有人根据事物id进行数据同步，保证一致性，而上面的所有描述信息就是zxid的信息。**通过zxid来保持节点之间的信息完整性。**

在集群内部中，zxid是通过网络来进行传递的，如过网络卡顿导致Leader的值是3，而某个Follower是2呢？

 我们可以通过其它机器获取值之前，进行手动同步，例如：sync path

```shell
# 删除节点
delete path

# 强制删除节点
rmr path
```

**注意：**在临时的节点中不能创建子节点，如果待删除的节点中含有子节点，则不能删除该节点。但是可以使用rmr进行强制删除。

## 7. zookeeper数据保持一致

**上面的操作内容可以保证数据在多个机器上保持一致，那么数据是如何保持一致的呢？**

### 7.1 paxos协议

![](/Users/trigger/Documents/WorkSpace/Notes/Hadoop/zookeeper/zookeeperImgs/paxos.png)

paxos协议原理：

> Paxos 算法解决的问题是一个分布式系统如何就某个值（决议）达成一致。一个典型的场景是，在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点执行相同的操作序列，那么他们最后能得到一个一致的状态。为保证每个节点执行相同的命令序列，需要在每一条指令上执行一个“一致性算法”以保证每个节点看到的指令一致。一个通用的一致性算法可以应用在许多场景中，是分布式计算中的重要问题。因此从20世纪80年代起对于一致性算法的研究就没有停止过。节点通信存在两种模型：共享内存和消息传递（Messages passing）。Paxos 算法就是一种基于消息传递模型的一致性算法。

假如有5台机器，有两机器进行提议，第一个说将班费改为10元，这个时候就会问旁边的两机器（半数原理），是否同意提议，但是，这两机器不知道提议者要干嘛，返回结果以后，提议者才将提议发送所有机器。如果同一时间来一个zxid更大的提议，那么它会受理zxid更大的一个，旧事物则不会再管。

![](/Users/trigger/Documents/WorkSpace/Notes/Hadoop/zookeeper/zookeeperImgs/paxos2.png)

**注意：**如果进行某种提议，server5分别向server1和server4发送提议，此时zxid=1，server1同意了请求。然后server2也向server3，server4发送提议，server3同意了，此时zxid=2，这个时候server4就是关键。由于两个提议都没有达到半数以上的回复所以不能执行，但是zxid=2更大，故受理这个，再返回结果时，server5见到没有回复，于是重新发送提议，此时的zxid=3，server4更偏向了server5，这时server2做同样的操作，然后整个集群开始无限循环，锁住了。(**可以参考，主战派和主和派与领导人之间的会议。**)

### 7.2 zab协议

![](/Users/trigger/Documents/WorkSpace/Notes/Hadoop/zookeeper/zookeeperImgs/zab.png)

> **1.** Zab协议是为分布式协调服务 Zookeeper专门设计的，是 Zookeeper保证数据一致性的核心算法。Zab借鉴了 Paxos算法，但又不像 Paxos那样，是一种通用的分布式一致性算法。支持崩溃恢复 和 原子广播协议。
> **2.** 在 Zookeeper中主要依赖 Zab协议来实现数据一致性，基于该协议，zk实现了一种主备模型（即 Leader和 Follower模型）的系统架构来保证集群中各副本之间数据的一致性。主备系统架构模型指只有一台客户端（Leader）负责处理外部的写事务请求，然后 Leader客户端将数据同步到其他 Follower节点。

总结来说，zab协议就是paxos协议的改版，存在主从模式，整个集群的操作全部从主节点出发，而不是多提议者
保证数据的完整一致性。**只有Leader进行提议，其他的Follower则是进行附和，同意还是不同意。**

## 8. zookeeper的读写数据流程

### 8.1 写入Leader

![](/Users/trigger/Documents/WorkSpace/Notes/Hadoop/zookeeper/zookeeperImgs/写入leader.png)

1. client向zookeeper发送请求
2. leader收到后自己先同意
3. 继续发送请求给旁边的follower，是否同意提议(不告诉内容)
4. follower向leader发送确认信息
5. 如果决议同意票数超过一般，则leader直接跟其他同意请求的follower进行修改信息
6. 然后告诉其他的剩余follower直接同步数据就行

### 8.2 写入Follower

![](/Users/trigger/Documents/WorkSpace/Notes/Hadoop/zookeeper/zookeeperImgs/写入follower.png)

1. client向zookeeper发送请求
2. follower向leader转发该请求
3. leader接到请求后先自己同意，然后向该follower发送请求提议
4. follower发送同意的信息
5. leader与投票的follower进行信息修改
6. 修改完后follower向leader发送写完的信息
7. 然后leader收到后，接受信息的follower向client发送修改完的信息
8. leader让其他follower同步数据（**事先没有告诉它**）
9. 其他follower返回修改完成的信息

###  8.3 读操作

1. 在Client向Follwer 或 Observer 发出一个读的请求； 

2. Follwer 或 Observer 把请求结果返回给Client；

## 9. zookeeper监听

```shell
#能够提供监听功能的命令有
# 监听只有一次
stat path [watch]监控节点状态变化
get path [watch] 监听值的变化
ls2 path [watch]监控节点状态变化和子节点变化
ls path [watch] 监控子节点变化
```

* **stat监听：**对被监听的文件夹的值和删除该文件夹有反应，如果创建子文件夹，没有反应。
* **ls监听：**只能监听子节点的变化，也就是子节点的值发生改变和删除创建子节点等操作
* **ls2监听：**只能监听子节点的增加和删除，当前节点的删除，不能监听任何值
* **get监听：**只监听值



## 10. 集群的启停和数据恢复

```shell
#集群的启停命令
1）启动ZK服务: sh bin/zkServer.sh start
2）查看ZK服务状态: sh bin/zkServer.sh status
3）停止ZK服务: sh bin/zkServer.sh stop
4）重启ZK服务: sh bin/zkServer.sh restart
```

我们知道zookeeper是基于内存的，但是内存在每次关机后就会被释放，内容也都会消失。经过实验发现，zookeeper重启后数据还在，这是为啥呢？

**数据恢复：**

进入/data文件夹中找到version2，里面存放的就是数据，数据会以日志和快照的形式存在磁盘中，在启动的时候可以加载进来。其中log存放的是操作日志，如果恢复上次关闭的状态的话，根据log进行**重播**的效率是很低很慢的。snapshot是快照，把某个时间点的文件夹和值的状态都记录下来，然后再存进磁盘中，用它恢复的话速度快，但是容错低。如果是一直快照的话，很浪费资源，如果是1h快照一次的话，那么就会丢失其中的部分数据，所以最好的办法就是log和snapshot进行结合。snapshot恢复到某个时间点，然后根据log进行调整。

## 11. 节点损坏后集群leader的选举和数据的恢复

**当zookeeper集群中的Leader宕机后，会触发新的选举，选举期间，整个集群是没法对外提供服务的。直到选出新的Leader之后，才能重新提供服务。**

### 11.1 leader的选举

* **apochId：**是集群的统治id，每次leader变化都会增加，类似于我国朝代的更替
* **zxid：**是每个事物的id，id越大，代表节点的数据也就越新
* **myid：**是每个节点的编号

在选举时，先看apochId的大，越先进的时代，数据也就越新。然后再看zxid，类似于思想越进步，数据一样是最新，当有多个机器zxid相同时，再看他们的myid，myid越大的机器，被选中的概率也就越大。

### 11.2 选举完后数据的恢复

Zab协议有两种模式，它们分别是恢复模式（选主）和广播模式（同步）。

**广播模式：**简单来说就是，集群服务还是正常的，只要保持数据同步就行。

**恢复模式：**集群服务不正常，所以要先选主，然后再进行广播模式

> 当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，恢复模式不接受客户端请求，当领导者被选举出来，且过半的Follower完成了和leader的状态同步以后，恢复模式就结束了。剩下未同步完成的机器会继续同步，直到同步完成并加入集群后该节点的服务才可用。

**失败恢复的两种情况**

> 1.leader发送完毕提议就宕机了
>
> 2.leader接受完毕ack，但是没有commit提交就宕机了

**选举leader的要求**

> 1.首先是zxid最大的，可以防止数据丢失
>
> 2.这个leader不能存在未完成的提议

**leader选完之后的动作**

> 1.扫清障碍，保证所有的提议都被执行了（完成上一个leader的遗愿）
>
> 2.将落后的follower进行数据同步，并且加入到follower的队伍中

==**注意：进入恢复模式的时候，zookeeper是暂停对外服务的，新的leader会引导集群做上述的任务，只有当同步恢复的节点个数超过一半的时候才对外服务，其他未同步的节点继续保持同步即可。**==