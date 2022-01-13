### Redis常用数据结构

#### 字符串

`Redis`字符串命令用于管理`Redis`中的字符串值. 字符串在`Redis`中以`key-value`的形式存储在`redisDb`的`dict`中. 字符串key经过哈希函数计算之后作为`dict`的键, 只能是字符串类型. 字符串的value是`dict`的值类型为`OBJ_STRING`类型. 

当value为字符串类型为string类型的时候, 根据字符串的长短决定编码的方式, 分别是`OBJ_ENCODING_RAW`或者`OBJ_ENCODING_EMBSTR`. 当字符串为long类型的时候, 编码类型为`OBJ_ENCODING_INT`.

##### 关于过期时间

字符串可以设置超时时间信息, 基本单位为毫秒, 过期时间戳为当前的时间戳加上过期时长, 这个时间戳为有效时间戳的`ddl`. 超出这个时间戳则认为无效, 这个信息存储在`redisDb`的**过期字典**中,  字典的key是字符串key经过哈希函数计算后的值, value就是这个有效时间戳的`ddl`信息. 查询时候判断是否过期需要对比当前时间戳和这个过期时间戳的信息, 如果超时则表示当前信息无效.

##### 常用指令

1. 设置字符串

   ```bash
   set key value
   ```

   将key-value应用到数据库中, 如果key已经设置则会覆盖掉旧值. 如果设置的时候不指定`EX`或者`PX`参数, set命令会清除原有的过期时间.

   

   ```bash
   setnx key value
   ```

   当前仅当key不存在的时候设置value

   

   ```bash
   setex key seconds value
   ```

   设置超时时间, 单位秒

   

   ```bash
   psetex key millseconds value
   ```

   设置超时时间, 单位毫秒

   

   批量设置

   ```bash
   mset key value [key value ...]
   msetnx key value [key value ...]
   ```

2. 修改字符串

   如果只想对value进行修改,可以通过append指令进行处理:

   ```bash
   append key value
   ```

   将value添加到原来value的末尾, 仅仅到value的值为string类型的时候才可以添加, 如果是int类型就会报错.

   

   `setrange`用于设置value的部分子串, 设置的时候从指定偏移量位置处修改, 如果偏移量大于源字符串长度, 则中间使用`\x00`填充.

   ```bash
   setrange key offset value
   ```

   常用的计数器指令包括`incr,decr,incrby,decrby,incrbyfloat`. 格式为

   ```bash
   incr key
   decr key
   
   incrby key delta
   decrby key delta
   
   incrbyfloat key delta
   ```

3. 获取字符串

   + `get`指令

     ```bash
     get key
     ```

     用于获取指定key的值, 没有设置则返回nil

   + `getset`指令

     ```bash
     getset key value
     ```

     获取key的原value, 并设置新的value

   + `getrange`指令

     ```bash
     getrange key start end
     ```

     获取key的value值从start到end部分的内容, 这里的value只能是字符串, 如果是整数则会转换为字符串.

   + `strlen`指令

     ```bash
     strlen key
     ```

     获取指定key的value字符串长度

   + `mget`指令

     ```bash
     mget key [key ...]
     ```

     返回所有key的value

4. 位操作

   `Redis`提供了位设置, 操作, 统计等命令.

   + `setbit`

     设置指定比特位上的比特位

     ```bash
     $ setbit key offset value
     ```

   + `getbit`

     获取指定偏移量上的比特位

     ```bash
     $ getbit key offset
     ```

   + `bitpos`

     ```bash
     $ bitpos key bit [start [end]]
     ```

     将value看成一个字节数组, 从start位置开始, 查找第一个设置为bit的位置

   + `bitcount`

     ```bash
     $ bitcount key [start] [end] 
     ```

     计算指定key对应value范围内1的数量

   + `bittop`

     对一个或者多个key进行操作, 并将目标值保存到新的key上

     ```bash
     $ bittop op_name target_key key [key ...]
     ```

   + `bitfield`

     将value看做字节数组, 从offset位置开始进行操作(获取/增加/设置)操作.



#### 散列表

当value以散列表作为存储的时候, 称作散列类型, 与`redisDb`key-value的散列存储有区别.

##### 存储结构

`Redis`的散列表底层存储为哈希表或者压缩列表. 当表中数据比较小的时候, 采用压缩列表存储.

当满足一定的条件的时候, 存储会从压缩列表转换为哈希表, 但是不会有哈希表转化为压缩列表的存在.

这里指的条件为:

1. key-value结构所有键值对的字符串长度都小于 上限值`hash-max-ziplist-value`.
2. 散列对象保存的键值对小于上限值`hash-max-ziplist-entries`.

##### 常用指令

+ 设置类指令

  ```bash
  $ hset key field value
  
  $ hmset key1 field1 value1 key2 field2 value2
  
  $ hsetnx key field value
  ```

+ 读取类指令

  ```bash
  $ hexists key field # 检查field是否存在
  
  $ hget key field
  
  $ hmget key field1 field2
  
  # 获取key下所有数据
  $ hkeys key
  $ hvals key
  $ hgetall key
  
  # 获取key下field总数
  $ hlen key
  
  # 遍历散列表中的kv对
  $ hscan key cursor [pattern] [count]
  ```

+ 删除类命令

  ```bash
  $ hdel key field [field ...]
  ```

  当key下的所有数据都删除的时候, key也会被删除.

+ 自增指令

  ```bash
  $ hincrby key field increment
  $ hincrbyfloat key field increment
  ```

  

#### 列表

列表的底层存储为`quicklist`,下面主要讨论`Redis`关于阻塞式`push/pop`的实现.

实现的功能为: 当列表对象不存在的时候, 会阻塞客户端, 直到列表对象不为空 或者 阻塞时间超过 `timeout`为止.

```bash
$ blpop key [key ...] timeout
```

这里需要注意的是只有有一个列表不为空, 则会直接返回, 只有所有都为空的时候才会阻塞.

当阻塞超时的时候, 会返回给客户端一个`nil`. 



#### 集合

`Redis`的无序集合由`intset`或者字典实现.

##### 常用的集合运算

1. 交集

   + `sinter` 求多集合的交集

     ```bash
     $ sinter key [key ...]
     ```

   + `sinterstore`求多集合的交集并存储

     ```bash
     $ sinter dest key [key ...]
     ```

2. 并集

   + `sunion`求并集

     ```bash
     $ sunion key [key ...]
     ```

3. 差集

   + `sdiff`求差集

     ```bash
     $ sdiff key [key ...]
     ```

#### 

#### 有序集合

有序集合的底层存储使用**压缩列表**或者**字典+跳表**的方式进行存储. 当记录数量小于一定值的时候使用压缩列表存储.

##### 常用指令

```bash
# 添加指令
zadd key score member
# 删除指令
zrem key member [member ...]

# 基数指令 获取排序集合中元素数量
zcard key

# 数量指令 获取分数范围内元素数量
zcount key min max

# 计数器
zincrby key increment member

# 获取排名
zrank key number

# 获取排名
zscore key number

# 遍历
zscan key cursor [pattern] [count]
```



##### 附录:

`Redis`指令参考 http://www.redis.cn/commands.html