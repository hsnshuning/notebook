# raft协议
raft是一种共识协议，目的就是在一个集群中对一个状态达成一致。

## 概念
* 任期： 每个节点都有一个任期号，这个任期号在触发leader选举的时候递增。类似于一个逻辑时钟，根据任期号我们可以分辨过期数据。

### 角色
raft把各个节点分为leader，candidate，follower三种角色。
1. leader：领导者，一个集群里面只能有一个leader节点
2. candidate：候选人，参与leader选举的节点
3. follower：跟随者，只同步leader发来的数据，如果客户端把请求发到了follower，会重定向到leader上。

## leader选举
节点刚开始都为follower状态，某个follower的选举超时定时器到时间了，这个follower会发起新一轮选举。
1. follower递增任期号，转换为candidate并给自己投票，向其他节点发送投票请求，同时开启一个投票超时计时器。
2. 其他节点收到投票请求，遵循FIFO的规范，投票给第一个收到的投票请求的节点。
3. 当该candidate收到的投票数大于节点数的一半的时候，candidate成为leader，向其他节点发送appendEntries请求。
4. 当candidate收到其他节点发送的appendEntries请求，并且任期号大于等于当前candidate任期号，说明集群已经选出leader了，candidate转换为follower。
5. 当candidate的定时器超时还没有收到半数以上的投票或者其他节点的appendEntries请求，就把任期号+1继续发送投票请求。为防止多个candidate投票，每个candidate的票数都不超过节点数的一半，所以一直超时的问题，会重设超时时间的时候有一个随机范围，尽量减少这种情况的发生。

### leader选举限制
为了保证数据的安全性，不是所有的follower转换为candidate都会拿到选票的，投票给这个节点的原则是比较日志信息，日志信息比自己新就投票给它。通过对比日志的最后一个日志数据来决定，先看任期号，任期号大的数据新，如果任期号相同，索引号大的新。并且为了防止出现一提交的日志被新leader覆盖掉的情况，当leader选举成功后，在接收客户端的请求前，需要发送一个空日志，保证当前leader必须在任期内至少有一条日志被提交。

### follower处理选举请求过程
当follower接收到requestVote请求，会进行如下判断。
1. 首先判断日志是否比自己的新，如果不比自己的新就拒绝。
2. 再判断是否是在leader迁移的情况下，如果是的话就接收，并且如果当前任期没有给其他节点投票的话就投票给这个节点。
3. 前面两种情况都不是，说明这个节点日志比自己新，那为了防止是这个节点自己出问题了，还要看一下自己在一定时间内是否收到了leader的心跳，如果没收到的话再投票，收到的话就拒绝。

### 投票RPC
请求参数

|---|---|
|参数|解释|
|---|---|
|term|候选人的任期号|
|candidateid|候选人id|
|lastLogIndex||
## 日志复制
日志为逻辑日志，可以理解为binlog，包含三部分，index 日志索引号，term 任期号，command 执行的修改。有两个日志index，commitedIndex 已经成功复制到了哪个日志，appliedIndex 已经应用到了哪个。
1. 当收到一个客户端的修改请求时，leader先在自己的日志中添加一条日志，然后向其他节点广播AppendEntries时带上该日志，appendEntries同时也带有commitedIndex。  
2. follower收到AppendEntries后，在本地添加appendEntries中的日志，将到commitedIndex的日志都提交并响应leader。
3. 当leader收到半数以上的AppendEntries应答时，该日志进入了**成功复制**状态，commitedIndex移动到这个日志。
4. 到这一步这个命令算是提交完成了，发送给应用层，应用层更新状态机，这时appliedIndex与commitedIndex相同了。

### 日志数据不一致
当发生leader选举时很可能出现日志数据不一致的问题，可能出现的场景有，follower比leader的数据多，follower比leader的数据少，follower中的日志和leader中对应日志index存储数据不同。raft的解决方案为用leader中的日志覆盖follower中不一致的日志。nextIndex 下一个同步给该节点的日志索引，matchIndex 该节点最大日志索引。
1. 当一个新leader选出来之后，会为每个follower维护一个nextIndex和matchIndex，nextIndex是当前leader的最新的日志，matchIndex为0。
2. leader会通过appendEntries给follower同步日志，带上日志，日志index以及term。
3. follower接收到appendEntries，根据appendEntries中的日志index和term跟自己的最新日志和term比较。
    1. 最新日志如果与这个日志匹配，返回成功。
    2. 不匹配，
        1. 当前节点日志index比leader大
            1. 如果比任期号相同但最新日志index比leader的大，就直接删除自己节点的数据。
        2. 当前节点日志index比leader小
            1. 跟leader要丢失的日志，返回当前节点最新的日志index以及leader发来的最新index。


### 日志追加RPC
日志追加RPC实现了心跳和日志追加两个功能，这两个功能用的都是日志追加RPC。日志为空的PRC为心跳包。



## 集群节点变更
### 添加
往集群中添加一个节点，不会将新添加的节点立刻加入到集群中，等到新节点的数据追上进度才会加入到集群。数据同步给新节点时根据轮次同步，一轮同步一部分数据。轮数一般会限制一个最大值。

### 删除
1. 删除leader节点：
    * leader节点发出变更节点配置的命令，该命令被提交后leader下线，然后集群中follower超时进行选举。
2. 如果一个节点被移出集群后它自己不知道，接不到leader的心跳这个节点就会一直超时，term号也一直在加，会对集群产生干扰。
    * raft通过preVote机制解决这个问题。在节点发起选举前，先发起preVote，preVote超过半数才会真正发起选举。

### 注意点
1. 为了避免单个单个添加节点，