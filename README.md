### SixRedis 设计图：
```
                   SixRedis               
            +---------------------+
            |                     |
            |                     |         +--------------+
            |   +-------------+   |         |              |
  Write Request |             +-------------> Master Redis |
+--------------->             <-------------+              |
<---------------+   Key Meta  |   |         +------+-------+
            |   |             |   |                |
            |   | +---------+ |   |                |
  Read Request  | |  Local  | |   |                |
+---------------> | Storage | |   |         +------v-------+
<---------------+ |         | +------------->              |
            |   | +---------+ <-------------+ Slave Redis  |
            |   +-------------+   |         |              |
            |                     |         +--------------+
            |                     |
            +---------------------+
                 proxy / embed

```

### 核心想法：
* 将远程的Redis内存内容同步到本地内存中
* 只同步自己关心的key内容
* 提供并发访问能力
* 提供淘汰机制
* 降低本地访问的延时
* 降低后端的压力

### 细节：
* local storage提供一个redis的内存实例，能够伪装为redis客户端，同步目标redis的数据
* 将redis的读写分为写和读操作，写操作同步到目标master redis中，并将相关的key记录到key meta中
* master redis执行写操作，同步到slave redis中，slave redis中的数据同步到local storage中
* local storage根据自己关心的key，选择保存数据还是丢弃数据
* 对于读数据，如果在local storage里面查到，并且是完全体，则直接返回，对非完全体，部分操作交由slave redis操作
* 对于不存在的key数据，将相关的key记录到key meta中，通过slave进行访问
* key meta保持在内存和本地磁盘中，提供重启恢复使用
* SixRedis可以embed使用，也可以作为单独的代理使用，当SixRedis重启时，加载Key meta数据
* 提供Proxy指令，格式为proxy ip, port [sync]， 开启新的redis同步，能够指定多个redis做slave，key不允许冲突
* 指定sync后，SixRedis伪装为redis客户端做slave of操作，不指定为增量同步，只同步新的数据
* 提供mode指令，格式为mode conn smart|master|slave 登记连接模式，master全部代理到master上，slave全部代理到slave上，smart为上述逻辑
* 提供多线程并发操作能力，锁key

### 用法：
* embed 使用，极端降低访问延时，极端提高并发操作，降低后端压力
* proxy 使用，显著降低延时，显著提高并发能力，降低后端压力, 兼容其他语言
* slave redis， 替代普通的redis做slave，显著提高并发能力，更加合理的使用内存
* master redis， 替代普通的redis做master，显著提高并发能力，更加合理的使用内存
* 全链路， 提供增强型redis协议，支持安全的异步操作