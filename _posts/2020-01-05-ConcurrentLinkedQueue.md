---
layout: post
title:  "ConcurrentLinkedQueue读书笔记"
date:   2020-01-05 15:51:06 +0800
categories: notes
---

实现一个线程安全的队列有二种方式：一种是使用阻塞算法，一种是使用非阻塞算法。Doug Lea实现的ConcurrentLinkedQueue是使用非阻塞的方式来实现的基于链接节点的无界线程安全队列，采用CAS算法是Michael & Scott改良后的。

ConcurrentLinkedQueue类图如图1所示，继承AbstractQueue，拥有一个内部类Node。

<p align="center">  
  <img src="https://tva1.sinaimg.cn/large/006tNbRwly1galgnsafo7j30n70d0aay.jpg" width="650" height="350"/>
  <p align="center">图1 ConcurrentLinkedQueue类图</p>
</p>

## 1. 入队列

入队列的过程就是将节点加入到队列尾的过程。主要过程如下：

* 将新节点设置成当前队列尾节点的下一个节点
* 更新尾指针，如果tail的next节点不为空，则将入队节点设置成tail。如果tail的next为空，则将入队节点设置成tail的next节点。所以tail并不总是最后一个节点，

**下面具体分析下源码：**

```Java
/** 源码来自Java11
 * 首先定位出尾节点，之后使用CAS算法设置入队节点为尾节点，失败重试。
 */ 
public boolean offer(E e) {
  	// 构造新节点
  	ConcurrentLinkedQueue.Node<E> newNode = 
    	new ConcurrentLinkedQueue.Node(Objects.requireNonNull(e));
  	ConcurrentLinkedQueue.Node<E> t = this.tail; 
  	ConcurrentLinkedQueue.Node p = t;

    do {
        while(true) {
            ConcurrentLinkedQueue.Node<E> q = p.next;
            // 如果尾节点next为null，则直接尝试将newNode设置成tail
            if (q == null) {
              	break;
            }
          	// 如果p==q，则队列为空
            if (p == q) {
              	p = t != (t = this.tail) ? t : this.head;
            } else {
                // 如果tail更新后使用新的tail的值，tail没有被其他线程更新的话，
                // 就直接让p移动到p的下一个节点的位置。以此找到正确的tail。
              	// 这里使用到了Java的语言特性t != (t = this.tail)是从左往右执行的，右边的t值会被
                // 更新。
             		p = p != t && t != (t = this.tail) ? t : q;
            }
        }
    } while(!NEXT.compareAndSet(p, (Void)null, newNode));// CAS设置p的next
  
  	/**
  	 *	p!=t的情况发生在两种情况下：
  	 * 条件1，由于tail是有条件更新的，当tail指向最后最后一个节点的时候，此时添加节点后，
  	 * t和p都没有变动，不会触发tail的更新,此时tail就是倒数第二个节点，等再次添加的时候，
  	 * 进入p!=t的条件，开始更新tail的位置。隔一个节点更新一次。
  	 *
     * 条件2，tail被其他线程更新了，p指向的tail，已经不是tail指向的位置了。
  	 */
    if (p != t) {
      	// weakCompareAndSet()硬件层面实现与compareAndSet()相同，都是调用同一段代码，
        // 但是weakCompareAndSet()可能虚拟机实现
      	TAIL.weakCompareAndSet(this, t, newNode); // 更新尾节点
    }
    return true; // offer()只会返回true。
}
```

由于没有使用锁只是单纯的使用CAS操作，所以虽然代码量很少，但是比较考验功底，需要仔细推敲。这里的实现是Michael & Scott改良后的代码，tail的定义并不是字面意思上的最后尾节点，而是距离尾节点很近的一个节点，这样可以省去一些CAS更新tail的操作，当然如果并发比较高的情况`p!=t`这个条件很容以达到，效果可能退化到每次都需要更新。

```java
do{
    while(true){
        ……
        p = p != t && t != (t = this.tail) ? t : q;
    }	
}while()
```

上面这个片段是添加节点算法的最重要的片段，像是一种竞速的思想，如果有其他线程更新了tail那么就使用从tail的位置向后遍历，否则按部就班的遍历。如果是多线程的情况下，就好像几个人在跑步，其中一个人跑到终点后打开了传送门，所有人都能直接到终点，然后继续跑，不断的产生终点。

##2.  出队列

出队列就是返回头节点的过程，主要过程如下：

* 如果head节点中有元素，直接弹出head节点里的元素，不更新head节点。
* 当head节点里没有元素是，出队操作才会更新head节点。

**下面具体分析源码：**

```java
/** 源码来自Java11
 * 
 */ 
public E poll() {
    label33:
    while(true) {
        ConcurrentLinkedQueue.Node<E> h = this.head;

        ConcurrentLinkedQueue.Node p;
        ConcurrentLinkedQueue.Node q;
        Object item;
        /*
         * (item = p.item) == null为true表示当前p指向的不为头节点了
         * !p.casItem(item, (Object)null)为true表示当前p指向的是头节点，并且取值成功，修改值为
         * null成功
         * 这里有一个短路原则在里面
         */
        for(p = h; (item = p.item) == null || !p.casItem(item, (Object)null); p = q) {
            // 当前队列为空，则直接返回null
            if ((q = p.next) == null) {
                this.updateHead(h, p);
                return null;
            }

            // 自引用了，则重新寻找队列头节点
            if (p == q) {
                continue label33;
            }
        }
		// CAS成功标志当前节点以及从链表中移除
        if (p != h) {
            this.updateHead(h, (q = p.next) != null ? q : p);
        }

        return item;
    }
}

/**
 * CAS更新头节点
 */
final void updateHead(ConcurrentLinkedQueue.Node<E> h, ConcurrentLinkedQueue.Node<E> p) {
    if (h != p && HEAD.compareAndSet(this, h, p)) {
        NEXT.setRelease(h, h); // 将原来的头节点自己指向自己
    }

}
```

