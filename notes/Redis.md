## Redis基本介绍
- Redis是NoSQL数据库，不是传统的关系型数据库。官网：https://redis.io/ 和 https://www.redis.cn/
- Redis: REmote Dictionary Server(远程字典服务器)，Redis性能非常高，单机能够达到15w qps,通常适合做缓存，也可以持久化。
- 是完全开源免费的，高性能的（key/value）分布式内存数据库，基于内存运行并支持持久化的NoSQL数据库, 是最热门的NoSQL数据库之一, 也称为数据结构服务器
### Redis的安装
- 下载后直接解压就有Redis的服务器端程序(redis-server.exe)和客户端程序(redis-cli.exe, 用来做测试的), 直接双击即可使用, 无需安装
- 指令 http://redisdoc.com/
- Redis服务端的核心组件监听端口, 收到指令后解析指令, 并做相应处理
### 基本使用
- Redis 安装好后，默认有16个数据库，初始默认使用0号库, 编号是0...15
    - 添加key-val [set k v]
    - 查看当前redis的 所有key [keys*]
    - 获取key对应的值 [get k]
    - 切换redis数据库 [select 索引号]
    - 如何查看当前数据库的key-val数量 [dbsize]
    - 清空当前数据库的key-val和清空所有数据库的key-val[flushdb/flushall], 前者清空当前数据库, 后者清空16个数据库的所有值
### CRUD
- Redis五大数据类型, String, Hash, List, Set, zset(有序集合)
- String
    - redis最基本的数据类型
    - string类型是二进制安全的, 除了普通的字符串, 也可以存放图片等数据
    - redis中字符串value最大是512M
    - set[如果存在就是修改], get, del
    - setex k 10 v, 设置后10秒销毁数据
    - mset同时set多个k-v
    - mget同时获取多个k-v
- Hash类似Golang中的Map, k-v的集合
    - hset/hget/hgetall/hdel
    - hmset/hmget/hlen/hexist
- List
    - 类别是简单的字符串, 按照插入顺序排序, 可以在头/尾
    - List本质是个链表, 元素有序且可以重复
    - lpush/rpush/lrange/lpop/rpop/del
    - lrange 链表名 开始索引 结束索引, 返回这个区间内的所有元素, 索引允许负数
    - lindex按照下标获取元素
    - LLEN 链表名, 返回长度, 不存在返回0
    - 如果值全删掉了, 对应的链表名也就消失了
- Set是String类型的无序集合
    - 底层是HashTable, 无序且不能重复
    - sadd, smembers, sismember, srem

### Golang操作Redis
- 1 ) 使用第三方开源的redi库: github.com/garyburd/redigo/redis
- 2 ) 在使用Redis前，先安装第三方Redis库，在GOPATH路径下执行安装指令: go get github.com/garyburd/redigo/redis
- 具体操作看API吧, redis.Dial等

### Redis链接池
- 说明: 可以通过Golang 对Redis操作， 还可以通过Redis链接池, 流程如下：
    - 1 ) 事先初始化一定数量的链接，放入到链接池
    - 2 ) 当Go需要操作Redis时，直接从 Redis 链接池取出链接即可。
    - 3 ) 这样可以节省临时获取 Redis 链接的时间，从而提高效率.