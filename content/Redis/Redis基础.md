+++
date = '2025-09-06T20:47:53+08:00'
draft = false
title = 'Redis基础'
+++

redis 是一款内存高速缓存数据库，支持丰富的数据类型。数据储存结构是 key-value 型的。

## Redis 的数据类型

Redis 所有的 key 都是字符串。一下讨论的储存值 value 的数据类型

### String

 String类型是二进制安全的，意思是 redis 的 string 可以包含任何数据。如数字，字符串，jpg图片或者序列化的对象。

```bash
# 使用命令有
set key_name value # 设置
get key_name # 获取值
del key_name # 删除
incr key_name # value + 1
incrby key_name num # value + num
decr key_name # value - 1
decrby key_name num # value - num
```

### List 列表

List 就是双端链表

```bash
lpush list_name value1 ... # 向名为 list_name（就是 key ） 链表的左端1存放 value
rpush list_name value1 ...
lpop list_name # 从 key_name 的左端弹出数据
rpop list_name
lrange list_name start stop # 获取链表指定范围的数据
lindex list_name index # 获取链表指定位置的数据
```

### Set 集合

Set 集合是 String 类型的无序集合，不能出现重复数据。集合是通过哈希表实现的，所以添加，删除，查找1的复杂度是O(1)

```bash
sadd set_name value1 ... # 向集合中添加元素
scaed set_name # 获取集合中的个数
smembers set_name # 返回集合中的元素
```

### Hash 散列

是一个 string 类型的字段和 value 值的映射表

```bash
hset hash_name field1 value1 # 向 hash_name 添加键值对
hget hash_name field # 获取 field 对应的值
hgetall hash_name # 获取散列表中包含的所有键值对
hdel hash_name field # 删除对应键值对
```

### Zset

是 Redis 中的有序集合，且不允许有重复元素。每个元素通过关联一个分数来对元素进行排序的

```bash
zadd zset_name score value1 ... # 添加一个元素
zrange zset_name start stop # 获取对应范围的元素
zrem zset_name value # 删除对应的元素
```

### HyperLogLogs

用来统计基数，即两个集合并集中不重复元素的估算个数，并不是准确的数值。

> 统计一个大型网站的每日访问 IP 数，就可以用此数据结构统计

```bash
pfadd key1 a b c d # 创建一组元素
pfcount key1 # 统计几个基数数量
pfadd key2 c d e f # 第二组元素
pfmerge key3 key1 key2 # 将 key1 和 key2 合并为 k3
pfcount k3
```

### Bitmap

即位图数据结构，用来操作二进制位来进行记录，只有 0 和 1 两个状态

> 可以用来统计用户登录情况（登录、未登录）因为每位状态只需要 1bit，365 天才需要 46 个字节左右，所以能很好的节省内存

```bash
setbit key_name offset value # 创建一组数据
getbit key_name offset # 查看 offset 值
bitcount key_name # 统计 value 为 0 的个数
```

### geospatial

推算地理位置1信息：两地之间的距离，方圆几里的人

```bash
# 添加地理位置
# 添加的是经纬度
# 有效的经度从-180度到180度
# 有效的纬度从-85.05112878度到85.05112878度
geoadd china:city 144.05 22.52 shengzhen 120.16 30.24 hangzhou 

geopos china:city shengzhen hangzhou # 获取指定成员的经纬度

# 获取两地距离
# m
# km
# mi 英里
# ft 英尺
geodist china:city shengzhen hangzhou m 

georadius china:city 110 30 1000 km # 以 100,30 这个坐标为中心, 寻找半径为1000km的城市
georadiusbymember china:city taiyuan 1000 km # 显示与指定成员一定半径范围内的其他成员
```

### Stream

是一种轻量级 MQ ，它的设计考虑了

- 消息ID的序列化生成
- 消息遍历
- 消息的阻塞和非阻塞读取
- 消息的分组消费
- 未完成消息的处理
- 消息队列监控
- ...

```bash
Stream 的结构
            
              Stream directin——>
            ID1 --> ID2 --> ID3 --> ID4 --> ID5 --> ID6 --> ID7 --> ID8 --> ID9 --> ID10
               |                 
              \|/
          consumer group    ----> consumer pending_ids[]
         last_delivered_id         

1. consumer group 是消费者组，一个消费者组可以有多个消费者
2. last_delivered_id 是游标，每个消费组会有个游标 last_delivered_id，任意一个消费者读取了消息都会使游标往前移动
3. pending_ids 是消费者(Consumer)的状态变量，作用是维护消费者的未确认（ack）的 id
```

操作

```bash
# * 表示服务器自动生成 ID，执行 xadd 自动创建一个消息队列并追加消息 
xadd key * field value
xdel key ID # 删除指定消息队列消息

xrange key - + # 获取消息队列。- 表示最小值，+ 表示最大值
xrange key id1 id2 # 获取指定 ID 序列的消息队列
xlen key # 获取消息队列长度
```

redis 允许在没有消费者组的情况下独立消费，即将 Stream 当成一个普通的 list 队列使用

```bash
# 从 Stream 头部读取两条消息
xread count 2 streams key 0-0
# 从Stream尾部读取一条消息，毫无疑问，这里不会返回任何消息
xread count 1 streams key $
# 从尾部阻塞等待新消息到来，下面的指令会堵住，直到新消息到来
# block 0表示永远阻塞，直到消息到来，block 1000表示阻塞1s，如果1s内没有任何消息到来，就返回nil
xread block 0 count 1 streams key $
```

消费者组

```bash
# 创建消费者组
xgroup create stream_list consumer_group1 0-0  #  表示从头开始消费
xgroup create stream_list consumer_group2 $  #  表示从尾部开始消费
xinfo stream stream_list # 获取 Stream 信息
xinfo groups stream_list

# 消费
# >号表示从当前消费组的last_delivered_id后面开始读
# 每当消费者读取一条消息，last_delivered_id变量就会前进
# 在 consumer_group1 消费组创建 consumer1 消费者消费 stream_list 队列中的数据
xreadgroup group consumer_group1 consumer1 count 1 stream stream_list > 
xreadgroup group consumer_group1 consumer1 block 0 count 1 streams codehole > # 阻塞
xinfo consumers stream_list consumer_group1 #  查看消费者状态
xpending stream_list consumer_group1 # 查看 pending 队列的数据
xack stream_list consumer_group1 ID # ack 消费者中的 pending 队列的数据
```

> redis 生成的消息 ID 由两部分组成，时间戳-序号。时间戳是毫秒级单位，是生成消息的Redis服务器时间，它是个64位整型。序号是在这个毫秒时间点内的消息序号，它也是个 64 位整型。Redis生成的ID是单调递增有序的。由于ID中包含时间戳部分，为了避免服务器时间错误而带来的问题，Redis的每个Stream类型数据都维护一个 latest_generated_id 属性，用于记录最后一个消息的ID。若发现当前时间戳退后（小于latest_generated_id所记录的），则采用时间戳不变而序号递增的方案来作为新消息 ID，从而保证ID的单调递增性质。
>
> 消费组每次读取数据后都会将数据放入 Pending 中，并不会立刻处理，而是要调用 xack 命令才会完成消息的处理。这样可以防止消费者崩溃导致消息丢失
