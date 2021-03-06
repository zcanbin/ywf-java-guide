## jdk的实现方式

**思路**：另启一个服务，利用jdk并发工具来控制唯一资源，如在服务中维护一个concurrentHashMap，其他服务对某个key请求锁时，通过该服务暴露的端口，以网络通信的方式发送消息，服务端解析这个消息，将concurrentHashMap中的key对应值设为true，分布式锁请求成功，[demo中写了一个基于netty通信的分布式锁](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FLiuWillow%2Fdistributed-lock%2Ftree%2Fmaster%2Fsingle-lock-server%2Fsrc%2Fmain%2Fjava%2Fcom%2Flwl%2Fserver)，当然你想用java的bio、nio或者整合dubbo、spring cloud feign来实现通信也没问题

```
 缺点：demo中的锁写的非常简单，但是要考虑可用性、可靠性、效率、扩展性的话，编码难度会比较高
```



## redis的实现方式

**原理**：利用redis的setnx的特性，能够保证同一时间内只有一个请求能够对相同的key执行setnx指令，我们可以理解为只有一个请求能够抢到这个锁，由于redis是单线程的，可以非常简单地实现这个功能。但是这里的锁的概念与我们平时锁理解的有一点区别，如mysql中的互斥锁，行数据被锁定以后，其他任何线程都无法对其进行删改操作，但是redis里只保证setnx的操作有这个特性，其他请求依然可以在key被锁住的情况下，执行del和set等操作，下面是redis分布式锁的各种实现方式和缺点，按照时间的发展排序：

- 1、直接setnx
   直接利用setnx，执行完业务逻辑后调用del释放锁，简单粗暴
   
   ```
   缺点：如果setnx成功，还没来得及释放，服务挂了，那么这个key永远都不会被获取到
   ```

- 2、setnx设置一个过期时间
   为了改正第一个方法的缺陷，我们用setnx获取锁，然后用expire对其设置一个过期时间，如果服务挂了，过期时间一到自动释放
   
   ```
   缺点：setnx和expire是两个方法，不能保证原子性，如果在setnx之后，还没来得及expire，服务挂了，还是会出现锁不释放的问题
   ```
   
- 3、set nx px
   redis官方为了解决第二种方式存在的缺点，在2.8版本为set指令添加了扩展参数nx和ex，保证了setnx+expire的原子性，使用方法：
   `set key value ex 5 nx` 
   
   ```
   缺点：
   ①如果在过期时间内，事务还没有执行完，锁提前被自动释放，其他的线程还是可以拿到锁
   ②上面所说的那个缺点还会导致当前的线程释放其他线程占有的锁
   ```

- 4、加一个事务id
   上面所说的第一个缺点，没有特别好的解决方法，只能把过期时间尽量设置的长一点，并且最好不要执行耗时任务
   第二个缺点，可以理解为当前线程有可能会释放其他线程的锁，那么问题就转换为保证线程只能释放当前线程持有的锁，即setnx的时候将value设为任务的唯一id，释放的时候先get key比较一下value是否与当前的id相同，是则释放，否则抛异常回滚，其实也是变相地解决了第一个问题
   
   ```
   缺点：get key和将value与id比较是两个步骤，不能保证原子性
   ```

- 5、set nx px + 事务id + lua
   我们可以用lua来写一个getkey并比较的脚本，jedis/luttce/redisson对lua脚本都有很好的支持
   
   ```
   缺点：集群环境下，对master节点申请了分布式锁，由于redis的主从同步是异步进行的，master在内存中写入了nx之后直接返回，客户端获取锁成功，此时master节点挂了，并且数据还没来得及同步，另一个节点被升级为master，这样其他的线程依然可以获取锁
   ```

- 6、redlock
   为了解决上面提到的redis集群中的分布式锁问题，redis的作者antirez的提出了red lock的概念，假设集群中所有的n个master节点完全独立，并且没有主从同步，此时对所有的节点都去setnx，并且设置一个请求过期时间re和锁的过期时间le，同时re必须小于le（可以理解，不然请求3秒才拿到锁，而锁的过期时间只有1秒也太蠢了），此时如果有n / 2 + 1个节点成功拿到锁，此次分布式锁就算申请成功
   
   ```
   **缺点**：可靠性还没有被广泛验证，并且严重依赖时间，好的分布式系统应该是异步的，并不能以时间为担保，程序暂停、系统延迟等都可能会导致时间错误
   ```
   
   

## 基于zookeeper实现的分布式锁

- 1、利用临时节点特性
   zookeeper的临时节点有两个特性，一是节点名称不能重复，二是会随着客户端退出而销毁，因此直接将key作为节点名称，能够成功创建的客户端则获取成功，失败的客户端监听成功的节点的删除事件
   
   ```
   缺点：所有客户端监听同一个节点，但是同时只有一个节点的事件触发是有效的，造成资源的无效调度
   ```
   
   
   
- 2、利用顺序临时节点特性
   zookeeper的顺序临时节点拥有临时节点的特性，同时，在一个父节点下创建创建的子临时顺序节点，会根据节点创建的先后顺序，用一个32位的数字作为后缀，我们可以用key创建一个根节点，然后每次申请锁的时候在其下创建顺序节点，接着获取根节点下所有的顺序节点并排序，获取顺序最小的节点，如果该节点的名称与当前添加的名称相同，则表示能够获取锁，否则监听根节点下面的处于当前节点之前的节点的删除事件，如果监听生效，则回到上一步重新判断顺序，直到获取锁。



## 总结

- 基于jdk的并发工具自己实现的锁
   
   ```
   优点：不需要引入中间件，架构简单
   缺点：编写一个可靠、高可用、高效率的分布式锁服务，难度较大
   ```
   
   
   
- redis set px nx + 唯一id + lua脚本
   
   ```
   优点：redis本身的读写性能很高，因此基于redis的分布式锁效率比较高
   缺点：依赖中间件，分布式环境下可能会有节点数据同步问题，可靠性有一定的影响，如果发生则需要人工介入
   ```
   
   
   
- 基于redis的redlock
   
   ```
   优点：可以解决redis集群的同步可用性问题
   缺点：依赖中间件，并没有被广泛验证，维护成本高，需要多个独立的master节点；需要同时对多个节点申请锁，降低了一些效率
   ```
   
   
   
- 基于zookeeper的分布式锁
   
   ```
   优点：不存在redis的超时、数据同步（zookeeper是同步完以后才返回）、主从切换（zookeeper主从切
   换的过程中服务是不可用的）的问题，可靠性很高
   缺点：依赖中间件，保证了可靠性的同时牺牲了一部分效率（但是依然很高）
   ```
   
   