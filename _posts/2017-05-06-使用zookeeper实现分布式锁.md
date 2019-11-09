---
layout:     post
title:      使用zookeeper实现分布式锁
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 杂谈
---
#### 应用场景
分布式锁主要用于在分布式环境中保护跨进程、跨主机、跨网络的共享资源实现互斥访问，以达到保证数据的一致性。

#### 架构分析
![分布式锁架构图.png](https://upload-images.jianshu.io/upload_images/10574922-bdfbffd37e84a9bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


左边的整个区域表示一个Zookeeper集群，locker是Zookeeper的一个持久节点，node_1、node_2、node_n是locker这个持久节点下面的临时顺序节点。右边3个client绿色的代表3台客户端服务器，server表示需要互斥访问的共享资源。

#### 实现思路和流程
1、在zookeeper指定节点（locker）下创建临时顺序节点node_n。  
2、获取locker下所有子节点children。  
3、对子节点按节点自增序号从小到大排序。  
4、判断本节点是不是第一个子节点，若是，则获取锁；若不是，则等待。  
5、使用zookeeper感知节点的功能，对本节点的上一个节点进行感知. 
 
6、当上一个节点被删除了，zookeeper会通知该线程，该线程就结束等待，并获取锁。  
7、释放锁，并删除该临时节点。

#### 具体实现代码
下面就具体使用java和zookeeper实现分布式锁，操作zookeeper使用的是apache提供的zookeeper的包。

***分布式锁类***
```
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

/**
 * Lock：Java的锁接口，并实现该接口的抽象方法
 * Watcher：zookeeper的节点感知接口，并实现process方法，当节点有改变时，会调用该方法
 */
public class DistributedLock  implements Lock, Watcher {


    private ZooKeeper zk = null;
    // 根节点
    private String ROOT_LOCK = "/locker";
    // 竞争的资源
    private String lockName;
    // 等待的前一个锁
    private String WAIT_LOCK;
    // 当前锁
    private String CURRENT_LOCK;
    // 同步计数器
    private CountDownLatch countDownLatch;
    private int sessionTimeout = 30000;

    /**
     * 配置分布式锁
     * @param config 连接的url
     * @param lockName 竞争资源
     */
    public DistributedLock(String config, String lockName) {
        this.lockName = lockName;
        try {
            // 连接zookeeper
            zk = new ZooKeeper(config, sessionTimeout, this);
            Stat stat = zk.exists(ROOT_LOCK, false);
            if (stat == null) {
                // 如果根节点不存在，则创建根节点
                zk.create(ROOT_LOCK, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }

    // 节点感知器，感知到节点变化会调用该方法
    public void process(WatchedEvent event) {
        System.out.println("感知节点变化类型："+event.getType().name());
        if (this.countDownLatch != null) {
            //如果同步计数器不为null，则减一
            this.countDownLatch.countDown();
        }
    }

    public void lock() {

        try {
            if (this.tryLock()) {
                System.out.println(Thread.currentThread().getName() + " " + lockName + "获得了锁");
                return;
            } else {
                // 等待锁
                waitForLock(WAIT_LOCK, sessionTimeout);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }

    public boolean tryLock() {
        try {
            String splitStr = "_lock_";
            if (lockName.contains(splitStr)) {
                throw new RuntimeException("锁名有误");
            }
            // 创建临时有序节点
            CURRENT_LOCK = zk.create(ROOT_LOCK + "/" + lockName + splitStr, new byte[0],
                    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
            System.out.println(CURRENT_LOCK + " 已经创建");
            // 取所有子节点
            List<String> subNodes = zk.getChildren(ROOT_LOCK, false);
            // 取出所有lockName的锁
            List<String> lockObjects = new ArrayList<String>();
            for (String node : subNodes) {
                String _node = node.split(splitStr)[0];
                if (_node.equals(lockName)) {
                    lockObjects.add(node);
                }
            }
            Collections.sort(lockObjects);
            System.out.println(Thread.currentThread().getName() + " 的锁是 " + CURRENT_LOCK);
            // 若当前节点为最小节点，则获取锁成功
            if (CURRENT_LOCK.equals(ROOT_LOCK + "/" + lockObjects.get(0))) {
                return true;
            }

            // 若不是最小节点，则找到自己的前一个节点
            String prevNode = CURRENT_LOCK.substring(CURRENT_LOCK.lastIndexOf("/") + 1);
            WAIT_LOCK = lockObjects.get(Collections.binarySearch(lockObjects, prevNode) - 1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
        return false;
    }

    public boolean tryLock(long timeout, TimeUnit unit) {
        try {
            if (this.tryLock()) {
                return true;
            }
            return waitForLock(WAIT_LOCK, timeout);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }

    // 等待锁
    private boolean waitForLock(String prev, long waitTime) throws KeeperException, InterruptedException {

        //获取并感知上一个节点
        Stat stat = zk.exists(ROOT_LOCK + "/" + prev, true);

        if (stat != null) {
            System.out.println(Thread.currentThread().getName() + "等待锁 " + ROOT_LOCK + "/" + prev);
            //初始化同步计数器，计数为1，当同步计数器为0，主线程才会向下执行
            this.countDownLatch = new CountDownLatch(1);
            // 计数等待，若等到前一个节点消失，则precess中进行countDown，停止等待，获取锁
            this.countDownLatch.await(waitTime, TimeUnit.MILLISECONDS);
            this.countDownLatch = null;
            System.out.println(Thread.currentThread().getName() + " 等到了锁");
        }
        return true;
    }

    public void unlock() {
        try {
            System.out.println("释放锁 " + CURRENT_LOCK);
            zk.delete(CURRENT_LOCK, -1);
            CURRENT_LOCK = null;
            zk.close();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }

    public Condition newCondition() {
        return null;
    }


    public void lockInterruptibly() throws InterruptedException {
        this.lock();
    }
    
}
```
***测试类***
```
public class Test {

    static int n = 500;

    public static void secskill() {
        System.out.println(--n);
    }

    public static void main(String[] args) {

        Runnable runnable = new Runnable() {
            public void run() {
                DistributedLock lock = null;
                try {
                    lock = new DistributedLock("127.0.0.1:2181", "node");
                    lock.lock();
                    secskill();
                    System.out.println(Thread.currentThread().getName() + "正在运行");
                } finally {
                    if (lock != null) {
                        lock.unlock();
                    }
                }
            }
        };

        for (int i = 0; i < 10; i++) {
            Thread t = new Thread(runnable);
            t.start();
        }
    }
}
```


