---
layout: post
title: "分布式锁"
date: 2017-10-15
tag: "work"
detail: 分布式锁是什么，作用是什么，有几种实现方式，具体实现。
img:

---

* content
{:toc}

## 分布式锁是什么

目前大多数网站及应用都是分布式部署的，我们为了保证数据的最终一致性，需要很多的技术方案来支持，比如分布式事务、分布式锁等。

为了实现多个线程在一个时刻同一个代码块只能有一个线程可执行，那么需要在某个地方做个标记，这个标记必须每个线程都能看到，当标记不存在时可以设置该标记，其余后续线程发现已经有标记了则等待拥有标记的线程结束同步代码块取消标记后再去尝试设置标记。这个标记可以理解为分布式锁。


## 分布式锁作用


* 当在分布式模型下，数据只有一份（或有限制），此时需要利用锁的技术控制某一时刻修改数据的进程数。

## 分布式锁的几种实现方式

    基于数据库表
    
1. 基于表的数据操作锁，当我们要加锁是，创建一条记录；释放锁时，删除这条记录。

2. 基于数据库排他锁，利用数据库自带的排他锁机制。select for update。

该方案强依赖于数据库的可用性。数据库是单点，一旦挂掉，将导致整个业务不可用。（1,2）

该锁不是阻塞的。依赖insert操作，一旦失败直接返回。没有获得锁的线程不会进入队列，要想获取锁，只能再次发起insert。（1）

天生是阻塞的。for update失败会一直处于阻塞状态。数据库宕机，则锁自动释放（2）

该锁没有失效时间。一旦delete失败，锁就永远存在了。（1）

该锁不是可重入锁。（1,2）


    
    基于缓存实现

    
    
    
    
    
    基于Zookeeper实现
    
基于Zookeeper临时有序节点实现分布式锁。 

大体原理：当每个客户端对某个方法加锁时，在Zookeeper的对应目录下，会生成一个唯一的瞬态的有序的znode。 

判断是否获得锁，只要判断自己是不是目录下最小的znode。 

如果不是，注册对比自己小的znode的监听。当比自己小的znode删除时，收到消息，判断自己是不是最小的。

当某个客户端获取锁后，突然挂掉。没关系，对应的znode会被自动删除，不会出现锁释放不了的问题。 

是阻塞锁 

可重入 

集群部署

使用zookeeper还是比较靠谱的，而且可以直接使用第三方包Curator，这个包直接封装了一个可重入的锁服务。

```aidl

public boolean tryLock(long timeout, TimeUnit unit) throws Exception {
        return interProcessMutex.acquire(timeout, unit);
}
public void unlock() throws Exception {
        interProcessMutex.release();
}

```
Curator的interProcessMutex.acquire获取锁，release释放锁。

只是性能上要比基于缓存差一点。 

但是生产环境上还是推荐zookeeper。

## Redis 分布式锁存在的问题和解决方案

    分布式锁存在的问题

均可能存在多进程拥有锁的情况。redis锁主要是expire时间与代码执行时间的问题，zk锁的问题在于zk是通过心跳监控进程存活状态，如果进程进行GC pause或者因为网络原因导致很长时间没与zk联系，则将导致zk认为进程已挂，而后锁自动释放，而此时进程并未挂任然在执行。
Redlock锁的时间问题。由于redis的expire的实现是通过pexpireat，如果某个节点发生时钟跳跃，则该节点可能过早释放锁导致一系列问题。


    解决方案

获取锁时提供一个fencing token(两种说法，一种说需要有序，一种说随机值就可以，我觉得随机值就可以)，在进程获取锁后对数据进行操作时，数据所在的资源服务器需要去锁中查看当前token，如果token对的才执行，不对则放弃执行。
我觉得对于放弃执行的应该在我们的代码块中增加类似事物的rollback的操作。因此如果资源服务器拒绝了我们的操作则表明此时起码已经存在了另外一个进程拥有锁了，为了保证数据安全性不能继续执行，因此需要回滚到执行代码块之前而继续去竞争锁。
至于Redis锁的时间问题，Antirez说在运维层面是可以控制时钟跳跃的区间的，只要能控制跳跃区间与expire的比例就没问题，详细可看《基于Redis的分布式锁真的安全吗？》