## 面试题
分库分表之后，id 主键如何处理？

## 面试官心理分析
其实这是分库分表之后你必然要面对的一个问题，就是 id 咋生成？因为要是分成多个表之后，每个表都是从 1 开始累加，那肯定不对啊，需要一个**全局唯一**的 id 来支持。所以这都是你实际生产环境中必须考虑的问题。

## 面试题剖析
### 基于数据库的实现方案
#### 数据库自增 id
这个就是说你的系统里每次得到一个 id，都是往一个库的一个表里插入一条没什么业务含义的数据，然后获取一个数据库自增的一个 id。拿到这个 id 之后再往对应的分库分表里去写入。

这个方案的好处就是方便简单，谁都会用；**缺点就是单库生成**自增 id，要是高并发的话，就会有瓶颈的；如果你硬是要改进一下，那么就专门开一个服务出来，这个服务每次就拿到当前 id 最大值，然后自己递增几个 id，一次性返回一批 id，然后再把当前最大 id 值修改成递增几个 id 之后的一个值；但是**无论如何都是基于单个数据库**。

**适合的场景**：你分库分表就俩原因，要不就是单库并发太高，要不就是单库数据量太大；除非是你**并发不高，但是数据量太大**导致的分库分表扩容，你可以用这个方案，因为可能每秒最高并发最多就几百，那么就走单独的一个库和表生成自增主键即可。

#### 设置数据库 sequence 或者表自增字段步长
可以通过设置数据库 sequence 或者表的自增字段步长来进行水平伸缩。

比如说，现在有 8 个服务节点，每个服务节点使用一个 sequence 功能来产生 ID，每个 sequence 的起始 ID 不同，并且依次递增，步长都是 8。

![database-id-sequence-step](/images/database-id-sequence-step.png)

**适合的场景**：在用户防止产生的 ID 重复时，这种方案实现起来比较简单，也能达到性能目标。但是服务节点固定，步长也固定，将来如果还要增加服务节点，就不好搞了。

### UUID
好处就是本地生成，不要基于数据库来了；不好之处就是，UUID 太长了、占用空间大，**作为主键性能太差**了；更重要的是，UUID 不具有有序性，会导致 B+ 树索引在写的时候有过多的随机写操作（连续的 ID 可以产生部分顺序写），还有，由于在写的时候不能产生有顺序的 append 操作，而需要进行 insert 操作，将会读取整个 B+ 树节点到内存，在插入这条记录后会将整个节点写回磁盘，这种操作在记录占用空间比较大的情况下，性能下降明显。

适合的场景：如果你是要随机生成个什么文件名、编号之类的，你可以用 UUID，但是作为主键是不能用 UUID 的。

```java
UUID.randomUUID().toString().replace(“-”, “”) -> sfsdf23423rr234sfdaf
```

### 获取系统当前时间
这个就是获取当前时间即可，但是问题是，**并发很高的时候**，比如一秒并发几千，**会有重复的情况**，这个是肯定不合适的。基本就不用考虑了。

适合的场景：一般如果用这个方案，是将当前时间跟很多其他的业务字段拼接起来，作为一个 id，如果业务上你觉得可以接受，那么也是可以的。你可以将别的业务字段值跟当前时间拼接起来，组成一个全局唯一的编号。

### snowflake 算法
snowflake 算法是 twitter 开源的分布式 id 生成算法，采用 Scala 语言实现，是把一个 64 位的 long 型的 id，1 个 bit 是不用的，用其中的 41 bit 作为毫秒数，用 10 bit 作为工作机器 id，12 bit 作为序列号。

- 1 bit：不用，为啥呢？因为二进制里第一个 bit 为如果是 1，那么都是负数，但是我们生成的 id 都是正数，所以第一个 bit 统一都是 0。
- 41 bit：表示的是时间戳，单位是毫秒。41 bit 可以表示的数字多达 `2^41 - 1`，也就是可以标识 `2^41 - 1` 个毫秒值，换算成年就是表示69年的时间。
- 10 bit：记录工作机器 id，代表的是这个服务最多可以部署在 2^10台机器上哪，也就是1024台机器。但是 10 bit 里 5 个 bit 代表机房 id，5 个 bit 代表机器 id。意思就是最多代表 `2^5`个机房（32个机房），每个机房里可以代表 `2^5` 个机器（32台机器）。
- 12 bit：这个是用来记录同一个毫秒内产生的不同 id，12 bit 可以代表的最大正整数是 `2^12 - 1 = 4096`，也就是说可以用这个 12 bit 代表的数字来区分**同一个毫秒内**的 4096 个不同的 id。

```
0 | 0001100 10100010 10111110 10001001 01011100 00 | 10001 | 1 1001 | 0000 00000000
```

```java
public class IdWorker {

    private long workerId;
    private long datacenterId;
    private long sequence;

    public IdWorker(long workerId, long datacenterId, long sequence) {
        // sanity check for workerId
        // 这儿不就检查了一下，要求就是你传递进来的机房id和机器id不能超过32，不能小于0
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(
                    String.format("worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        if (datacenterId > maxDatacenterId || datacenterId < 0) {
            throw new IllegalArgumentException(
                    String.format("datacenter Id can't be greater than %d or less than 0", maxDatacenterId));
        }
        System.out.printf(
                "worker starting. timestamp left shift %d, datacenter id bits %d, worker id bits %d, sequence bits %d, workerid %d",
                timestampLeftShift, datacenterIdBits, workerIdBits, sequenceBits, workerId);

        this.workerId = workerId;
        this.datacenterId = datacenterId;
        this.sequence = sequence;
    }

    private long twepoch = 1288834974657L;

    private long workerIdBits = 5L;
    private long datacenterIdBits = 5L;

    // 这个是二进制运算，就是 5 bit最多只能有31个数字，也就是说机器id最多只能是32以内
    private long maxWorkerId = -1L ^ (-1L << workerIdBits);

    // 这个是一个意思，就是 5 bit最多只能有31个数字，机房id最多只能是32以内
    private long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);
    private long sequenceBits = 12L;

    private long workerIdShift = sequenceBits;
    private long datacenterIdShift = sequenceBits + workerIdBits;
    private long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
    private long sequenceMask = -1L ^ (-1L << sequenceBits);

    private long lastTimestamp = -1L;

    public long getWorkerId() {
        return workerId;
    }

    public long getDatacenterId() {
        return datacenterId;
    }

    public long getTimestamp() {
        return System.currentTimeMillis();
    }

    public synchronized long nextId() {
        // 这儿就是获取当前时间戳，单位是毫秒
        long timestamp = timeGen();

        if (timestamp < lastTimestamp) {
            System.err.printf("clock is moving backwards.  Rejecting requests until %d.", lastTimestamp);
            throw new RuntimeException(String.format(
                    "Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }

        if (lastTimestamp == timestamp) {
            // 这个意思是说一个毫秒内最多只能有4096个数字
            // 无论你传递多少进来，这个位运算保证始终就是在4096这个范围内，避免你自己传递个sequence超过了4096这个范围
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0;
        }

        // 这儿记录一下最近一次生成id的时间戳，单位是毫秒
        lastTimestamp = timestamp;

        // 这儿就是将时间戳左移，放到 41 bit那儿；
        // 将机房 id左移放到 5 bit那儿；
        // 将机器id左移放到5 bit那儿；将序号放最后12 bit；
        // 最后拼接起来成一个 64 bit的二进制数字，转换成 10 进制就是个 long 型
        return ((timestamp - twepoch) << timestampLeftShift) | (datacenterId << datacenterIdShift)
                | (workerId << workerIdShift) | sequence;
    }

    private long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    private long timeGen() {
        return System.currentTimeMillis();
    }

    // ---------------测试---------------
    public static void main(String[] args) {
        IdWorker worker = new IdWorker(1, 1, 1);
        for (int i = 0; i < 30; i++) {
            System.out.println(worker.nextId());
        }
    }

}

```

怎么说呢，大概这个意思吧，就是说 41 bit 是当前毫秒单位的一个时间戳，就这意思；然后 5 bit 是你传递进来的一个**机房** id（但是最大只能是 32 以内），另外 5 bit 是你传递进来的**机器** id（但是最大只能是 32 以内），剩下的那个 12 bit序列号，就是如果跟你上次生成 id 的时间还在一个毫秒内，那么会把顺序给你累加，最多在 4096 个序号以内。

所以你自己利用这个工具类，自己搞一个服务，然后对每个机房的每个机器都初始化这么一个东西，刚开始这个机房的这个机器的序号就是 0。然后每次接收到一个请求，说这个机房的这个机器要生成一个 id，你就找到对应的 Worker 生成。

利用这个 snowflake 算法，你可以开发自己公司的服务，甚至对于机房 id 和机器 id，反正给你预留了 5 bit + 5 bit，你换成别的有业务含义的东西也可以的。

这个 snowflake 算法相对来说还是比较靠谱的，所以你要真是搞分布式 id 生成，如果是高并发啥的，那么用这个应该性能比较好，一般每秒几万并发的场景，也足够你用了。

 
 时钟回拨问题；
    趋势递增，而不是绝对递增；
    不能在一台服务器上部署多个分布式ID服务；

    第2点算不上缺点，毕竟如果绝对递增的话，需要牺牲不少的性能；第3点也算不上缺点，即使一台足够大内存的服务器，在部署一个分布式ID服务后，还有很多可用的内存，可以用来部署其他的服务，不一定非得在一台服务器上部署一个服务的多个实例；可以参考elasticsearch的主分片和副本的划分规则：某个主分片的副本是不允许和主分片在同一台节点上--因为这样意思不大，如果这个分片只拥有一个副本，且这个节点宕机后，服务状态就是"RED"；

所以这篇文章的主要目的是解决第1个缺点：时钟回拨问题；先来看一下sharding-jdbc的DefaultKeyGenerator.java中generateKey()方法的源码：

    @Override
    public synchronized Number generateKey() {
        long currentMillis = timeService.getCurrentMillis();
        Preconditions.checkState(lastTime <= currentMillis, "Clock is moving backwards, last time is %d milliseconds, current time is %d milliseconds", lastTime, currentMillis);
        if (lastTime == currentMillis) {
            if (0L == (sequence = ++sequence & SEQUENCE_MASK)) {
                currentMillis = waitUntilNextTime(currentMillis);
            }
        } else {
            sequence = 0;
        }
        lastTime = currentMillis;
        if (log.isDebugEnabled()) {
            log.debug("{}-{}-{}", new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date(lastTime)), workerId, sequence);
        }
        return ((currentMillis - EPOCH) << TIMESTAMP_LEFT_SHIFT_BITS) | (workerId << WORKER_ID_LEFT_SHIFT_BITS) | sequence;
    }

    说明：从这段代码可知，sharding-jdbc并没有尝试去解决snowflake算法时钟回拨的问题，只是简单判断如果lastTime <= currentMillis不满足就抛出异常：Preconditions.checkState(lastTime <= currentMillis, "Clock is moving backwards, last time is %d milliseconds, current time is %d milliseconds", lastTime, currentMillis);

改进思路

snowflake算法一个很大的优点就是不需要依赖任何第三方组件，笔者想继续保留这个优点：毕竟多一个依赖的组件，多一个风险，并增加了系统的复杂性。

snowflake算法给workerId预留了10位，即workId的取值范围为[0, 1023]，事实上实际生产环境不大可能需要部署1024个分布式ID服务，所以：将workerId取值范围缩小为[0, 511]，[512, 1023]这个范围的workerId当做备用workerId。workId为0的备用workerId是512，workId为1的备用workerId是513，以此类推……

    说明：如果你的业务真的需要512个以上分布式ID服务才能满足需求，那么不需要继续往下看了，这个方案不适合你^^；

改进实现

对generateKey()方法的改进如下：

    // 修改处: workerId原则上上限为1024, 但是为了每台sequence服务预留一个workerId, 所以实际上workerId上限为512
    private static final long WORKER_ID_MAX_VALUE = 1L << WORKER_ID_BITS >> 1;
        
    /**
      * 保留workerId和lastTime, 以及备用workerId和其对应的lastTime
      */
     private static Map<Long, Long> workerIdLastTimeMap = new ConcurrentHashMap<>();
     
    /**
     * Generate key. 考虑时钟回拨, 与sharding-jdbc源码的区别就在这里</br>
     * 缺陷: 如果连续两次时钟回拨, 可能还是会有问题, 但是这种概率极低极低
     * @return key type is @{@link Long}.
     * @Author 阿飞
     */
    @Override
    public synchronized Number generateKey() {
        long currentMillis = System.currentTimeMillis();
     
        // 当发生时钟回拨时
        if (lastTime > currentMillis){
            // 如果时钟回拨在可接受范围内, 等待即可
            if (lastTime - currentMillis < MAX_BACKWARD_MS){
                try {
                    Thread.sleep(lastTime - currentMillis);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }else {
                // 如果时钟回拨太多, 那么换备用workerId尝试
     
            // 当前workerId和备用workerId的值的差值为512
                long interval = 512L;
                // 发生时钟回拨时, 计算备用workerId[如果当前workerId小于512,
                // 那么备用workerId=workerId+512; 否则备用workerId=workerId-512, 两个workerId轮换用]
                if (MyKeyGenerator.workerId >= interval) {
                    MyKeyGenerator.workerId = MyKeyGenerator.workerId - interval;
                } else {
                    MyKeyGenerator.workerId = MyKeyGenerator.workerId + interval;
                }
     
                // 取得备用workerId的lastTime
                Long tempTime = workerIdLastTimeMap.get(MyKeyGenerator.workerId);
                lastTime = tempTime==null?0L:tempTime;
                // 如果在备用workerId也处于过去的时钟, 那么抛出异常
                // [这里也可以增加时钟回拨是否超过MAX_BACKWARD_MS的判断]
                Preconditions.checkState(lastTime <= currentMillis, "Clock is moving backwards, last time is %d milliseconds, current time is %d milliseconds", lastTime, currentMillis);
                // 备用workerId上也处于时钟回拨范围内的逻辑还可以优化: 比如摘掉当前节点. 运维通过监控发现问题并修复时钟回拨
            }
        }
     
        // 如果和最后一次请求处于同一毫秒, 那么sequence+1
        if (lastTime == currentMillis) {
            if (0L == (sequence = ++sequence & SEQUENCE_MASK)) {
                currentMillis = waitUntilNextTime(currentMillis);
            }
        } else {
            // 如果是一个更近的时间戳, 那么sequence归零
            sequence = 0;
        }
     
        lastTime = currentMillis;
        // 更新map中保存的workerId对应的lastTime
        workerIdLastTimeMap.put(MyKeyGenerator.workerId, lastTime);
     
        if (log.isDebugEnabled()) {
            log.debug("{}-{}-{}", new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date(lastTime)), workerId, sequence);
        }
        System.out.println(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date(lastTime))
                    +" -- "+workerId+" -- "+sequence+" -- "+workerIdLastTimeMap);
        return ((currentMillis - EPOCH) << TIMESTAMP_LEFT_SHIFT_BITS) | (workerId << WORKER_ID_LEFT_SHIFT_BITS) | sequence;
    }

改进验证


    第一个红色箭头的地方通过修改本地时间，通过启动了备用workerId从而避免了时钟回拨60s（window系统直接修改系统时间模拟）引起的问题；第二个红色箭头在60s内再次模拟时钟回拨，就会有问题，因为无论是workerId还是备用workerId都会有冲突；如果第二红色箭头是60s后模拟时钟回拨，依然可以避免问题，原因嘛，你懂得；

再次优化

每个workerId可以配置任意个备用workerId，由使用者去平衡sequence服务的性能及高可用，终极版代码如下：

    public final class MyKeyGenerator implements KeyGenerator {
     
        private static final long EPOCH;
        
        private static final long SEQUENCE_BITS = 12L;
        
        private static final long WORKER_ID_BITS = 10L;
        
        private static final long SEQUENCE_MASK = (1 << SEQUENCE_BITS) - 1;
        
        private static final long WORKER_ID_LEFT_SHIFT_BITS = SEQUENCE_BITS;
        
        private static final long TIMESTAMP_LEFT_SHIFT_BITS = WORKER_ID_LEFT_SHIFT_BITS + WORKER_ID_BITS;
     
        /**
         * 每台workerId服务器有3个备份workerId, 备份workerId数量越多, 可靠性越高, 但是可部署的sequence ID服务越少
         */
        private static final long BACKUP_COUNT = 3;
     
        /**
         * 实际的最大workerId的值<br/>
         * workerId原则上上限为1024, 但是需要为每台sequence服务预留BACKUP_AMOUNT个workerId,
         */
        private static final long WORKER_ID_MAX_VALUE = (1L << WORKER_ID_BITS) / (BACKUP_COUNT + 1);
     
        /**
         * 目前用户生成ID的workerId
         */
        private static long workerId;
        
        static {
            Calendar calendar = Calendar.getInstance();
            calendar.set(2018, Calendar.NOVEMBER, 1);
            calendar.set(Calendar.HOUR_OF_DAY, 0);
            calendar.set(Calendar.MINUTE, 0);
            calendar.set(Calendar.SECOND, 0);
            calendar.set(Calendar.MILLISECOND, 0);
            // EPOCH是服务器第一次上线时间点, 设置后不允许修改
            EPOCH = calendar.getTimeInMillis();
        }
        
        private long sequence;
        
        private long lastTime;
     
        /**
         * 保留workerId和lastTime, 以及备用workerId和其对应的lastTime
         */
        private static Map<Long, Long> workerIdLastTimeMap = new ConcurrentHashMap<>();
     
        static {
            // 初始化workerId和其所有备份workerId与lastTime
            // 假设workerId为0且BACKUP_AMOUNT为4, 那么map的值为: {0:0L, 256:0L, 512:0L, 768:0L}
            // 假设workerId为2且BACKUP_AMOUNT为4, 那么map的值为: {2:0L, 258:0L, 514:0L, 770:0L}
            for (int i = 0; i<= BACKUP_COUNT; i++){
                workerIdLastTimeMap.put(workerId + (i * WORKER_ID_MAX_VALUE), 0L);
            }
            System.out.println("workerIdLastTimeMap:" + workerIdLastTimeMap);
        }
     
        /**
         * 最大容忍时间, 单位毫秒, 即如果时钟只是回拨了该变量指定的时间, 那么等待相应的时间即可;
         * 考虑到sequence服务的高性能, 这个值不易过大
         */
        private static final long MAX_BACKWARD_MS = 3;
     
        /**
         * Set work process id.
         * @param workerId work process id
         */
        public static void setWorkerId(final long workerId) {
            Preconditions.checkArgument(workerId >= 0L && workerId < WORKER_ID_MAX_VALUE);
            MyKeyGenerator.workerId = workerId;
        }
        
        /**
         * Generate key. 考虑时钟回拨, 与sharding-jdbc源码的区别就在这里</br>
         * 缺陷: 如果连续两次时钟回拨, 可能还是会有问题, 但是这种概率极低极低
         * @return key type is @{@link Long}.
         * @Author 阿飞
         */
        @Override
        public synchronized Number generateKey() {
            long currentMillis = System.currentTimeMillis();
     
            // 当发生时钟回拨时
            if (lastTime > currentMillis){
                // 如果时钟回拨在可接受范围内, 等待即可
                if (lastTime - currentMillis < MAX_BACKWARD_MS){
                    try {
                        Thread.sleep(lastTime - currentMillis);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }else {
                    tryGenerateKeyOnBackup(currentMillis);
                }
            }
     
            // 如果和最后一次请求处于同一毫秒, 那么sequence+1
            if (lastTime == currentMillis) {
                if (0L == (sequence = ++sequence & SEQUENCE_MASK)) {
                    currentMillis = waitUntilNextTime(currentMillis);
                }
            } else {
                // 如果是一个更近的时间戳, 那么sequence归零
                sequence = 0;
            }
     
            lastTime = currentMillis;
            // 更新map中保存的workerId对应的lastTime
            workerIdLastTimeMap.put(MyKeyGenerator.workerId, lastTime);
     
            if (log.isDebugEnabled()) {
                log.debug("{}-{}-{}", new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date(lastTime)), workerId, sequence);
            }
     
            System.out.println(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date(lastTime))
                    +" -- "+workerId+" -- "+sequence+" -- "+workerIdLastTimeMap);
            return ((currentMillis - EPOCH) << TIMESTAMP_LEFT_SHIFT_BITS) | (workerId << WORKER_ID_LEFT_SHIFT_BITS) | sequence;
        }
     
        /**
         * 尝试在workerId的备份workerId上生成
         * @param currentMillis 当前时间
         */
        private long tryGenerateKeyOnBackup(long currentMillis){
            System.out.println("try GenerateKey OnBackup, map:" + workerIdLastTimeMap);
     
            // 遍历所有workerId(包括备用workerId, 查看哪些workerId可用)
            for (Map.Entry<Long, Long> entry:workerIdLastTimeMap.entrySet()){
                MyKeyGenerator.workerId = entry.getKey();
                // 取得备用workerId的lastTime
                Long tempLastTime = entry.getValue();
                lastTime = tempLastTime==null?0L:tempLastTime;
     
                // 如果找到了合适的workerId
                if (lastTime<=currentMillis){
                    return lastTime;
                }
            }
     
            // 如果所有workerId以及备用workerId都处于时钟回拨, 那么抛出异常
            throw new IllegalStateException("Clock is moving backwards, current time is "
                    +currentMillis+" milliseconds, workerId map = " + workerIdLastTimeMap);
        }
        
        private long waitUntilNextTime(final long lastTime) {
            long time = System.currentTimeMillis();
            while (time <= lastTime) {
                time = System.currentTimeMillis();
            }
            return time;
        }
    }

    核心优化代码在方法tryGenerateKeyOnBackup()中，BACKUP_COUNT即备份workerId数越多，sequence服务避免时钟回拨影响的能力越强，但是可部署的sequence服务越少，设置BACKUP_COUNT为3，最多可以部署1024/(3+1)即256个sequence服务，完全够用，抗时钟回拨影响的能力也得到非常大的保障。

改进总结

这种改进方案最大优点就是没有引入任何第三方中间件（例如redis，zookeeper等），但是避免时钟回拨能力得到极大的提高，而且时钟回拨本来就是极小概率。阿飞Javaer认为这种方案能够达到绝大部分sequence服务的需求了；






一线大厂的分布式唯一ID生成方案


目录

    前言
    UUID
    mysql主键自增
    mysql多实例主键自增
    雪花算法
    redis生成方案
    总结
    悬念

前言

分布式系统中我们会对一些数据量大的业务进行分拆，如：用户表，订单表。因为数据量巨大一张表无法承接，就会对其进行分库分表。可以去看一下你知道怎么分库分表吗？如何做到永不迁移数据和避免热点吗？ 和

如何永不迁移数据和避免热点? 根据服务器指标分配数据量(揭秘篇)

但一旦涉及到分库分表，就会引申出分布式系统中唯一主键ID的生成问题，永不迁移数据和避免热点的文章中要求需要唯一ID的特性：

    1、整个系统ID唯一

    2、ID是数字类型，而且是趋势递增的

    3、ID简短，查询效率快

什么是递增？如：第一次生成的ID为12，下一次生成的ID是13，再下一次生成的ID是14。这个就是生成ID递增。

什么是趋势递增？如：在一段时间内，生成的ID是递增的趋势。如：再一段时间内生成的ID在【0，1000】之间，过段时间生成的ID在【1000，2000】之间。但在【0-1000】区间内的时候，ID生成有可能第一次是12，第二次是10，第三次是14。

那有什么方案呢？往下看
UUID

这个方案是小伙伴们第一个能过考虑到的方案.UUID 的核心思想是使用「机器的网卡、当地时间、一个随机数」来生成 UUID.使用 UUID 的方式只需要调用 UUID.randomUUID().toString() 就可以生成，这种方式方便简单，本地生成，不会消耗网络。

优点：

    1、代码实现简单。

    2、本机生成，没有性能问题

    3、因为是全球唯一的ID，所以迁移数据容易

缺点：

    1、每次生成的ID是无序的，无法保证趋势递增

    2、UUID的字符串存储，查询效率慢

    3、存储空间大

    4、ID本事无业务含义，不可读

应用场景：

    1、类似生成token令牌的场景

    2、不适用一些要求有趋势递增的ID场景

此UUID方案是不适用大部分的需求
 
mysql主键自增

这个方案就是利用了mysql的主键自增auto_increment，默认每次ID加1。

优点：

    1、数字化，id递增

    2、查询效率高

    3、具有一定的业务可读

缺点：

    1、存在单点问题，如果mysql挂了，就没法生成iD了

    2、数据库压力大，高并发抗不住

mysql多实例主键自增

这个方案就是解决mysql的单点问题，在auto_increment基本上面，设置step步长

你想了解一线大厂的分布式唯一ID生成方案吗？

 

每台的初始值分别为1,2,3...N，步长为N（这个案例步长为4）。设置初始值可以通过下面的 sql 进行设置：

    set @@auto_increment_offset = 1;     // 设置初始值
    set @@auto_increment_increment = 4;  // 设置步长

优点：解决了单点问题

缺点：一旦把步长定好后，就无法扩容；而且单个数据库的压力大，数据库自身性能无法满足高并发

    应用场景：数据不需要扩容的场景

此方案也不满足老顾的需求，因为不方便扩容（记住这个方案，嘿嘿）
![图片](https://user-images.githubusercontent.com/65522342/125912084-7932e914-4683-4e41-8f87-cbd14b2154cd.png)

 
批量申请自增 ID


可以解决无 ID 可分的问题，它的原理就是一次性给对应的数据库上分配一批的 id 值进行消费，使用完了，再回来申请

在设计的初始阶段可以设计一个有初始值字段，并有步长字段的表，当每次要申请批量 ID 的时候，就可以去该表中申请，每次申请后「初始值 = 上一次的初始值 + 步长」。

这样就能保持初始值是每一个申请的 ID 的最大值，避免了 ID 的重复，并且每次都会有 ID 使用，一次就会生成一批的 id 来使用，这样访问数据库的次数大大减少。

但是这一种方案依旧有自己的缺点，依然不能抗真正意义上的高并发。

 
雪花snowflake算法

这个算法网上介绍了很多，老顾这里就不详细介绍。雪花算法生成64位的二进制正整数，然后转换成10进制的数。64位二进制数由如下部分组成：

你想了解一线大厂的分布式唯一ID生成方案吗？

 ![图片](https://user-images.githubusercontent.com/65522342/125909841-5a217924-cd58-4bc4-942f-c07ec08b6ad3.png)


    1位标识符：始终是0
    41位时间戳：41位时间截不是存储当前时间的时间截，而是存储时间截的差值（当前时间截 - 开始时间截 )得到的值，这里的的开始时间截，一般是我们的id生成器开始使用的时间，由我们程序来指定的
    10位机器标识码：可以部署在1024个节点，如果机器分机房（IDC）部署，这10位可以由 5位机房ID + 5位机器ID 组成
    12位序列：毫秒内的计数，12位的计数顺序号支持每个节点每毫秒(同一机器，同一时间截)产生4096个ID序号

优点：

    1、此方案每秒能够产生409.6万个ID，性能快

    2、时间戳在高位，自增序列在低位，整个ID是趋势递增的，按照时间有序递增

    3、灵活度高，可以根据业务需求，调整bit位的划分，满足不同的需求

缺点：

    1、依赖机器的时钟，如果服务器时钟回拨，会导致重复ID生成

在分布式场景中，服务器时钟回拨会经常遇到，一般存在10ms之间的回拨；小伙伴们就说这点10ms，很短可以不考虑吧。但此算法就是建立在毫秒级别的生成方案，一旦回拨，就很有可能存在重复ID。

此方案暂不符合老顾的需求（嘿嘿，看看怎么优化这个方案，小伙伴们先记住）
redis生成方案

利用redis的incr原子性操作自增，一般算法为：

年份 + 当天距当年第多少天 + 天数 + 小时 + redis自增

第一种是使用 RedisAtomicLong 原子类使用 CAS 操作来生成 ID。

    @Service
    public class RedisSequenceFactory {
        @Autowired
        RedisTemplate<String, String> redisTemplate;
     
        public void setSeq(String key, int value, Date expireTime) {
            RedisAtomicLong counter = new RedisAtomicLong(key, redisTemplate.getConnectionFactory());
            counter.set(value);
            counter.expireAt(expireTime);
        }
     
        public void setSeq(String key, int value, long timeout, TimeUnit unit) {
            RedisAtomicLong counter = new RedisAtomicLong(key, redisTemplate.getConnectionFactory());
            counter.set(value);
            counter.expire(timeout, unit);
        }
     
        public long generate(String key) {
            RedisAtomicLong counter = new RedisAtomicLong(key, redisTemplate.getConnectionFactory());
            return counter.incrementAndGet();
        }
     
        public long incr(String key, Date expireTime) {
            RedisAtomicLong counter = new RedisAtomicLong(key, redisTemplate.getConnectionFactory());
            counter.expireAt(expireTime);
            return counter.incrementAndGet();
        }
     
        public long incr(String key, int increment) {
            RedisAtomicLong counter = new RedisAtomicLong(key, redisTemplate.getConnectionFactory());
            return counter.addAndGet(increment);
        }
     
        public long incr(String key, int increment, Date expireTime) {
            RedisAtomicLong counter = new RedisAtomicLong(key, redisTemplate.getConnectionFactory());
            counter.expireAt(expireTime);
            return counter.addAndGet(increment);
        }
    }

第二种是使用 redisTemplate.opsForHash() 和结合 UUID 的方式来生成生成 ID。

    public Long getSeq(String key,String hashKey,Long delta) throws BusinessException{
            try {
                if (null == delta) {
                    delta=1L;
                }
                return redisTemplate.opsForHash().increment(key, hashKey, delta);
            } catch (Exception e) {  // 若是redis宕机就采用uuid的方式
                int first = new Random(10).nextInt(8) + 1;
                int randNo=UUID.randomUUID().toString().hashCode();
                if (randNo < 0) {
                    randNo=-randNo;
                }
                return Long.valueOf(first + String.format("%16d", randNo));
            }
        }

 

优点：有序递增，可读性强

缺点：占用带宽，每次要向redis进行请求

整体测试了这个性能如下：

    需求：同时10万个请求获取ID
    1、并发执行完耗时：9s左右
    2、单任务平均耗时：74ms
    3、单线程最小耗时：不到1ms
    4、单线程最大耗时：4.1s

性能还可以，如果对性能要求不是太高的话，这个方案基本符合老顾的要求。

但不完全符合业务老顾希望id从 1 开始趋势递增。（当然算法可以调整为 就一个 redis自增，不需要什么年份，多少天等）。

 
总结

如果今天老顾就介绍以上几个方案，其实没有必要，网上多的是；

一线大厂的分布式id方案绝没有这个简单，他们对高并发，高可用的要求很高。

如redis方案中，每次都要去redis去请求，有网络请求耗时，并发强依赖了redis。这个设计是有风险的，一旦redis挂了，整个系统不可用。

而且一线大厂也会考虑到ID安全性的问题，如：redis方案中，用户是可以预测下一个id号是多少，因为算法是递增的。

这样的话竞争对手 第一天中午12点下个订单，就可以看到平台的订单id是多少，第二天中午12点再下一单，又平台订单id到多少。这样就可以猜到平台1天能产生多少订单了，这个是绝对不允许的，公司绝密啊。

 

 

目录

    前言
    改造数据库主键自增
    竞争问题
    突发阻塞问题
    双buffer方案
    总结

前言

上一篇文章中介绍了分布式唯一ID你想了解一线大厂的分布式唯一ID生成方案吗？ ，留了一个悬念，这里老顾就介绍一下两种大厂的方案思路。希望能够帮到大家。
改造数据库主键自增

老顾在前一篇文章中介绍了利用数据库的自增主键的特性，可以实现分布式ID；这个ID比较简短明了，适合做userId，正好符合如何永不迁移数据和避免热点? 根据服务器指标分配数据量(揭秘篇)文章中的ID的需求。但这个方案有严重的问题：

    1、一旦步长定下来，不容易扩容

    2、数据库压力山大

我们小伙伴们看看怎么优化这个方案，先看数据库压力大，为什么压力大？是因为我们每次获取ID的时候，都要去数据库请求一次。那我们可以不可以不要每次去取？

思路我们可以请求数据库得到ID的时候，可设计成获得的ID是一个ID区间段。



 ![图片](https://user-images.githubusercontent.com/65522342/125911962-0e08b87e-8da7-4871-a8e2-376587a7f360.png)


我们看上图，有张id规则表：

    1、id表示为主键，无业务含义。

    2、biz_tag为了表示业务，因为整体系统中会有很多业务需要生成ID，这样可以共用一张表维护

    3、max_id表示现在整体系统中已经分配的最大ID

    4、desc描述

    5、update_time表示每次取的ID时间

我们再来看看整体流程：

    1、【用户服务】在注册一个用户时，需要一个用户ID；会请求【生成ID服务(是独立的应用)】的接口

    2、【生成ID服务】会去查询数据库，找到user_tag的id，现在的max_id为0，step=1000

    3、【生成ID服务】把max_id和step返回给【用户服务】；并且把max_id更新为max_id = max_id + step，即更新为1000

    4、【用户服务】获得max_id=0，step=1000；

    5、 这个用户服务可以用ID=【max_id + 1，max_id+step】区间的ID，即为【1，1000】

    6、【用户服务】会把这个区间保存到jvm中

    7、【用户服务】需要用到ID的时候，在区间【1，1000】中依次获取id，可采用AtomicLong中的getAndIncrement方法。

    8、如果把区间的值用完了，再去请求【生产ID服务】接口，获取到max_id为1000，即可以用【max_id + 1，max_id+step】区间的ID，即为【1001，2000】

这个方案就非常完美的解决了数据库自增的问题，而且可以自行定义max_id的起点，和step步长，非常方便扩容。

而且也解决了数据库压力的问题，因为在一段区间内，是在jvm内存中获取的，而不需要每次请求数据库。即使数据库宕机了，系统也不受影响，ID还能维持一段时间。
竞争问题

以上方案中，如果是多个用户服务，同时获取ID，同时去请求【ID服务】，在获取max_id的时候会存在并发问题。

    如用户服务A，取到的max_id=1000 ;用户服务B取到的也是max_id=1000，那就出现了问题，Id重复了。那怎么解决？

其实方案很多，加分布式锁，保证同一时刻只有一个用户服务获取max_id。当然也可以用数据库自身的锁去解决。


![图片](https://user-images.githubusercontent.com/65522342/125910682-8f6f5048-7c90-4b56-b706-5f87627ee58e.png)

 

利用事务方式加行锁，上面的语句，在没有执行完之前，是不允许第二个用户服务请求过来的，第二个请求只能阻塞。
突发阻塞问题


![图片](https://user-images.githubusercontent.com/65522342/125910608-adb5be4b-ae6a-4958-9a75-6fac3eb628f7.png)

 

上图中，多个用户服务获取到了各自的ID区间，在高并发场景下，id用的很快，如果3个用户服务在某一时刻都用完了，同时去请求【ID服务】。因为上面提到的竞争问题，所有只有一个用户服务去操作数据库，其他二个会被阻塞。

    小伙伴就会问，有这么巧吗？同时id用完。我们这里举的是3个用户服务，感觉概率不大；如果是100个用户服务呢？概率是不是一下子大了。

出现的现象就是一会儿突然系统耗时变长，一会儿好了，就是这个原因导致的，怎么去解决？
双buffer方案

在一般的系统设计中，双buffer会经常看到，怎么去解决上面的问题也可以采用双buffer方案。



 ![图片](https://user-images.githubusercontent.com/65522342/125910563-25d006e4-7ec8-48a6-ab8d-20cd24d1b17d.png)


在设计的时候，采用双buffer方案，上图的流程：

    1、当前获取ID在buffer1中，每次获取ID在buffer1中获取

    2、当buffer1中的Id已经使用到了100，也就是达到区间的10%

    3、达到了10%，先判断buffer2中有没有去获取过，如果没有就立即发起请求获取ID线程，此线程把获取到的ID，设置到buffer2中。

    4、如果buffer1用完了，会自动切换到buffer2

    5、buffer2用到10%了，也会启动线程再次获取，设置到buffer1中

    6、依次往返

双buffer的方案，小伙伴们有没有感觉很酷，这样就达到了业务场景用的ID，都是在jvm内存中获得的，从此不需要到数据库中获取了。允许数据库宕机时间更长了。

因为会有一个线程，会观察什么时候去自动获取。两个buffer之间自行切换使用。就解决了突发阻塞的问题。
总结

此方案是美团公司使用的分布式ID算法，小伙伴们如果想了解更深，可以去网上搜下，老顾这里应该介绍了比较详细了。

    当然此方案美团还做了一些别的优化，监控id使用频率，自动设置步长step，从而达到对id节省使用。

此ID方案非常适合老顾前几篇文章中的id需求如何永不迁移数据和避免热点? 根据服务器指标分配数据量(揭秘篇) 。

但此ID存在一定的问题，就是太过连续，竞争对手可以预测，不适合订单ID。那还有没有别的方案。下次老顾介绍一下美团的另一种生成ID方案。谢谢！！！
