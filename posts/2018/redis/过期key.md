# redis key过期处理
redis如果给一个key设置过期时间，redis是怎么管理这些key的，又是如何在key过期时删除key的。

而redis又是单线程的，因此更应该避免redis在大部分时间都在轮询检查key是否过期造成线上读写卡死。

redis的设计比较巧妙，它设计了一个字典，将每个设置了过期时间的key都放入了这个字典中，然后做如下两件事：
1. 定时遍历这个字典来删除到期的key
2. 惰性删除策略，即当客户端访问到这个key时，检查这个key的过期时间，如果过期则立即删除。

惰性删除很好理解，那么定时遍历又是怎么做的，如何做到遍历又不造成客户端卡死

### 定时遍历
上面说到，redis会将所有设置了过期时间的key放入一个字典中，然后去遍历这个字典。

默认情况redis会每秒进行10次过期扫描，而这种扫描并不是去扫描整个字典，而是用了一种扫描策略，流程如下：
1. 从字典中随机取20个key
2. 删除这20个key中已经过期的key
3. 如果过期的key过期比率超过1/4，那就重复步骤1

另外为了避免扫描时间过长，导致客户端卡死的现象，redis给这个扫描时间加了个上限，默认下为25ms。

这里有个问题会出现：如果在同一个时间出现大批量key过期，会怎么样？  
这会造成一直在遍历key，不断的循环直到过期key变得稀疏。这直接就造成了线上请求卡顿，cpu飙升。   
因此为了避免这个问题，最好给过期时间加一个随机过期值。

这里举一个案例：  
很多公司会为一些特殊的节日举办一些活动，如中秋或者国庆活动，在这些活动结束之后，其实在redis里的数据都是冗余的，可以清除了。  
因此正常情况下，我们会给这些redis的key加一个过期时间。但是我们需要注意的是，如果我们的参与活动的用户量非常的庞大，那么这些key会在一个时间内同时过期，就会造成一定的卡顿现象。  
为了避免这种现象，我们最好给这些key的过期时间加一个随机值，就能避免了，把过期时间分散开。

### 从库的策略
从库更加简单，作为从库的redis是不会进行过期扫描的，主库在key到期时，会在aof文件里增加一条del命令，然后同步到从库。

但是这里就会有个问题，出库的同步指令可能没有那么及时的到达从库，然后又有人访问了这个key，这就导致了主从数据不一致了。