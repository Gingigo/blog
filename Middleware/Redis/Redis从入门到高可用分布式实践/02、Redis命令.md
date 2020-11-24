# Redis 命令

### 通用命令

- keys 

  - 意思：返回 redis所有匹配的 key ，返回list
  - 语法： keys pattern 
  - 例子： keys *
  - 注意：一般不在生产环境中的主节点中用，因为是一个O(n)的时间复杂度

- dbsize 

  - 意思：命令用于返回当前数据库的 key 的数量。返回 int
  - 语法： dbsize
  - 注意：时间复杂度 O(1),没有什么风险

- exists

  - 意思：判断 key 是否存在，返回 int
  - 语法：exists key  [key...]
  - 例子：exists t0
  - 注意：复杂度 O(1),没有什么风险

- del 

  - 意思：删除key，返回 int
  - 语法 :  del key [key...]
  - 例子： del t1 t2
  - 注意：时间复杂度O(1)

- expire 

  - 意思：设置 key 的过期时间
  - 语法：expire key seconds
  - 例子：expire t3 60 # 设置t3 30秒后过期
  - 注意：
    - 可以通过 “ttl key”命令，查看多少剩余过期的时间 
    - 可以通过 “persist key”命令，去掉过期
    - 时间复杂度O(1)

- type

  - 意思：获取数据类型
  - 语法：type key
  - 例子：type t4
  - 注意:  时间复杂度O(1)

### 单线程

**单线程为什么这么快？**

1. 纯内存
2. 非阻塞IO
3. 避免线程却换和竞态切换

**单线程需要注意什么？**

1. 一次只能执行一条命令
2. 拒绝长（慢）命令，如keys、flushall、flushdb、show lua script...

### 字符串

**值类型**

- 字符串，key=hello,value=world （常常用来存json 或者压缩的二进制数据)，最大可以存储 512MB
- 数字，key=count,value=1
- 位图（bitMap），key=bits，value=[1,0,1,1,1,0,0,0]

**场景**

- 缓存
- 计数器
- 分布式锁

**命令**

- get key 			# 获取 key 对应的 value，O(1)
- set key value   [ex soconds] [nx/xx]  # 设置 key-value，O(1) 
  - setnx key value (= set key value nx )	# key 不存在，才可以设置 ,O(1)
  - set key value xx                                        # key 存在，才可以设置 ,O(1)
  - set key value ex soconds                         #  设置 key-value，seconds 秒后过期,原子操作，O(1)
- del key              # 删除 key-value，O(1) 
- incr key             # key 增加1，如果 key 不存在，自增后 get(key) =1, O(1)
- decr key            # key 自减肥1，如果 key 不存在，自减后 get(key)=-1, O(1)
- incrby key k      # key 自增 k，如果 key 不存在，自增后 get(key)=k,O(1)
- decrby key k     # key 自减 k，如果 key 不存在，自减后 get(key)=-k,O(1) 

- mget key1 key2 key3             # 批量获取key，原子操作 ,O(n)
- mset key1 value1 key2 value2 #批量设置 key-value ，O(n)
- getset key newvalue  # 设置 key-newvalue 并返回旧的 value ，O(1)
- append key value      # 将value追加到旧的value ,O(1)
- strlen key                    # 返回字符串的长度（注意中文）,O(1)
- incrbyfloat key 3.5     # 增加 key 对应的值3.5  , O(1）
- getrange key start end  # 获取字符串指定下标所有的值，[start,end],O(1)
- setrange key index value # 设置指定下标所对应的值,O(1）

**实战**

- 统计用户访问页面的次数
- 缓存数据库返回的数据
- 分布式id (可以用 incr id)

### 哈希

<table>
    <td>key</td>
    <td>
    	<table>
            <tr>
            <td>field1</td>
            <td>value1</td>
            </tr>
            <tr>
            <td>field2</td>
            <td>value2</td>
            </tr>
        </table>
    </td>
</table>

**特点**

- Map（Map） ，Map中的Map
- “Small redis” ，小型的redis
- field 不能相同，value可以相同

**命令 （所有命令以 h 开头）**  

- hget key field								                   # 获取 hash key 对应的 field 的 value ，O(1)
- hset key field value                                         # 设置 hash key 对应的 field 的 value ，O(1)
- hdel key field                                                   # 删除 hash key 对应的 field 的 value，O(1)
- hexists key field                                              # 判断 hash key 是否有 field，O(1)
- hlen key                                                           # 获取 hash key field 的数量  ，O(1)
- hmget key f1  f2 ... fn                                     # 批量获取 hash key 的一批 f1  f2 ... fn  对应的值 ，O(n)
- hmset key  f1 v1 f2 v2 ... fn vn                      # 批量设置 hash key 的一批 f1 v1 f2 v2 ... fn vn ，O(n)
- hincrby / hincrbyfloat  key field num           # 给 hash key 中的 field 值增加 num ， O(1)
- hgetall key                                                       # 返回 hash key 对应的所有 field 和value，O(n)
- hvals key                                                          # 返回 hash key 对应的所有 field 的 value，O(n)
- hkeys key                                                         # 返回 hash key 对应的所有 field  ，O(n)
- hsetnx key field value                                # 设置 hash key 对应 field 的 value (如果 field 存在，则失败）O(1)

**实战**

- 记录用户访问页面的次数（hincrby key field ）
- 缓存mysql 查出的数据对象 (对象的属性 说就是 field，值就是 value)



**注意**

- 在使用的时候, hgetall key ,如果数据大的话，返回速度慢，注意 redis 是单线程的！

### 列表

<table>
    <td>key</td>
    <td>
    	<table>
            <tr>
            <td>value1</td>
            <td>value2</td>
            <td>value3</td>
            <td>value4</td>
            </tr>
        </table>
    </td>
</table>



**特点**

- 有序
- 可重复
- 左右两边插入弹出



**命令( 以 L 开头）**

- rpush key value1 value2...valueN                     # 从列表右端插入值 （1...N个），O(1~n)
- lpush key value1 value2...valueN                     # 从列表左端插入值 （1...N个），O(1~n)
- linsert key before|after value newValue        # 在key对应的list 中，第一个 value 在 前|后 插入 newValue ，O（n）
- lpop key                                                                # 从key 对应的 list 中，右侧弹出一个 item ，O(1)
- rpop key                                                               # 从key 对应的 list 中，左侧弹出一个 item ，O(1)
- lrem key count value                                          # 根据 count 值 从列表中删除所有value相等的值 ,O(n)
  - count > 0                                                       # 从左到右，删除最多 count 个 value 相等的项
  - count < 0                                                       # 从右到左，删除最多 count 个 value 相等的项
  - count = 0                                                       # 删除所有 value 相等的项
- ltrim key start end                                              # 按照索引范围修剪 list，保留范围 [start,end]，O(n)
- lrange key start end                                           # 获取列表指定索引范围所有 item ，范围 [start,end]，O(n)
- lindex key index                                                  # 获取列表指定索引的item ，按列表下标，O(n)
- llen key                                                                 # 获取列表的长度 ，O(1)
- lset key index newValue                                    # 设置列表指定索引值为newValue ，O(n)
- blpop  key timeout            # lpop 阻塞版本，timeout 是阻塞超时时间，timeout=0 为一直等待，O(1)
- brpop  key timeout            # rpop 阻塞版本，timeout 是阻塞超时时间，timeout=0 为一直等待，O(1)

**实战**

- 微博，关注人发布信息**时间轴**
- lpush+lpop = 栈
- lpush+ rpop = 队列
- lpush+ltrim = 固定大小容器
- lpush+brpop=消息队列



### 集合

<table>
    <td>key</td>
    <td>
    	<table>
            <tr>
                <td>value2</td>
                <td>value4</td>
            </tr>
            <tr>
                <td>value1</td>
                <td>value3</td>
            </tr>
        </table>
    </td>
</table>



**特点**

- 无序
- 无重复
- 支持集合间的操作 （交集、并集等）



**命令（所有命令以 s 开头）**

- sadd key element                                          # 向集合 key 添加 element (如果存在返回0),O(1)
- srem key element                                          # 删除集合key 中的 element (如果存在返回0),O(1)
- scard   key                                                       # 计算集合 key 的大小,O(1)
- sismember key element                               # 集合 key 是否存在 element,O(1)
- srandmember key num                                # 随机查出集合 key 中的 num 个 元素，不会改变集合大小
- spop key  [num]                                              # 从集合中随机弹出 num (默认一个)个元素，会改变集合大小
- smembers                                                       # 返回集合中的所有元素 ，数据是无序的，O(n)
- sdiff key1 key2                                                # 返回 集合 key1 减去 集合 key2 的差集
- sinter key1 key2                                              # 返回 集合 key1 与 集合 key2 的交集（相同的部分）
- sunion key1 key2                                            #  返回 集合 key1 与 集合 key2 的并集 (全部元素)



**实战**

- 抽奖系统 （sadd 添加用id，通过 spop / srandmember 抽奖）
- 点赞/踩 功能
- 给用户添加标签



### 有序集合

<table>
    <td>key</td>
    <td>
    	<table>
            <tr>
                <td>score4:1</td>
                <td>value4</td>
            </tr>
            <tr>
                <td>score2:99</td>
                <td>value2</td>
            </tr>
            <tr>
                <td>score3:100</td>
                <td>value3</td>
            </tr>
            <tr>
                <td>score1:201</td>
                <td>value1</td>
            </tr>
        </table>
    </td>
</table>



**特点**

- 无重复元素
- 有序
- 组成： element + score

**命令（命令以 z 开头）**

- zadd key score element                           # 向集合key 添加 score 和 element (可以多对),O(logN)
- zrem key element                                     # 删除集合 key 中元素为 element （可以多个),O(1)
- zscore key element                                   # 获取元素 element 的 分数,O(1)
- zincrby key increScore element             # 给element 增加 increScore 分数 ,O(1)
- zcard key                                                    # 返回有序集合的元素个数,O(1)
- zrank key element                                    # 查看元素 element 的排名
- zrange key start end  [withscores]         # 查看指定范围的升序元素[并带上分数]，范围[start,end] 
- zrangebyscore key  mminScore maxScore  [withscores] #返回指定分数范围内升序元素 ，范围[start,end] 
- zcount key mminScore maxScore          # 返回指定分数范围内有多少个元素
- zremrangebyrank key start end             # 删除指定排名内升序元素，范围[start,end] 

**实战**

- 排行榜


















