---
layout: post
title: "ConcurrentHashmap拾遗"
date: 2018-06-11 14:23:19 +0800
comments: true
categories: 2018-06
tags: [Java,集合]
---
### 为什么Hashtable ConcurrentHashmap不支持key或者value为null？

在很多java资料中，都有提到 ConcurrentHashmap HashMap和Hashtable都是key-value存储结构，但他们有一个不同点是 ConcurrentHashmap、Hashtable不支持key或者value为null，而HashMap是支持的。为什么会有这个区别？在设计上的目的是什么？

<!--more-->在网上找到了这样的解答:The main reason that nulls aren’t allowed in ConcurrentMaps (ConcurrentHashMaps, ConcurrentSkipListMaps) is that ambiguities that may be just barely tolerable in non-concurrent maps can’t be accommodated. The main one is that if map.get(key) returns null, you can’t detect whether the key explicitly maps to null vs the key isn’t mapped. In a non-concurrent map, you can check this via map.contains(key), but in a concurrent one, the map might have changed between calls.

理解如下：ConcurrentHashmap和Hashtable都是支持并发的，这样会有一个问题，当你通过get(k)获取对应的value时，如果获取到的是null时，你无法判断，它是put（k,v）的时候value为null，还是这个key从来没有做过映射（map）。HashMap是非并发的，可以通过contains(key)来做这个判断。而支持并发的Map在调用m.contains（key）和m.get(key),m可能已经不同了。

个人觉得这个解答还是很有道理的，也是解决了心头的一个疑惑，大牛们在设计时确实考虑的很多，在这里分享给大家。

类似的解答还有这个： 
down vote 
I believe it is, at least in part, to allow you to combine containsKey and get into a single call. If the map can hold nulls, there is no way to tell if get is returning a null because there was no key for that value, or just because the value was null.

Why is that a problem? Because there is no safe way to do that yourself. Take the following code:

```java
if (m.containsKey(k)) {
   return m.get(k);
} else {
   throw new KeyNotPresentException();
}12345
```

Since m is a concurrent map, key k may be deleted between the containsKey and get calls, causing this snippet to return a null that was never in the table, rather than the desired KeyNotPresentException.

Normally you would solve that by synchronizing, but with a concurrent map that of course won’t work. Hence the signature for get had to change, and the only way to do that in a backwards-compatible way was to prevent the user inserting null values in the first place, and continue using that as a placeholder for “key not found”.

[参考地址](http://stackoverflow.com/questions/698638/why-does-concurrenthashmap-prevent-null-keys-and-values%20%E5%8F%82%E8%80%83%E5%9C%B0%E5%9D%80)

其他辅助性（佐证性质）的理由见下文。（对于ConcurrentHashmap，我参考其他人的相关文章，觉得还有其他理由。）

---

### ConcurrentHashmap中get方法为何不需要加锁同步

对于1.6，我们知道ConcurrentHashmap的get方法是没有加锁的，**所以value不能为null刚好用上**：newEntry对象是通过 `new HashEntry(K k , V v, HashEntry next)` 来创建的。如果另一个线程刚好new 这个对象时，当前线程来get它。因为没有同步，就可能会出现当前线程得到的newEntry对象是一个没有完全构造好的对象引用。

回想一下我们之前讨论的**DCL**的问题，这里也一样，没有锁同步的话，new 一个对象对于多线程看到这个对象的状态是没有保障的，这里同样有可能一个线程new这个对象的时候还没有执行完构造函数就被另一个线程得到这个对象引用。
所以才需要判断一下：`if (v != null)` 如果确实是一个不完整的对象，则使用锁的方式再次get一次。（来自参考文章1）

```java
V get(Object key, int hash) {
        if (count != 0) { // read-volatile // ①
            HashEntry<K,V> e = getFirst(hash); 
            while (e != null) {
                if (e.hash == hash && key.equals(e.key)) {
                    V v = e.value;
                    if (v != null)  // ② 注意这里
                        return v;
                    return readValueUnderLock(e); // recheck 见参考文章2
                }
                e = e.next;
            }
        }
        return null;
}
```



对于1.7，应该是通过UNSAFE.putOrderedObject（保证刚刚插入表头的节点被读取）来保证；对于1.8，采用CAS和synchronized保证并发一致，但似乎没有做对正在put数据的二次获取，未仔细研究，以上见参考文章3。

参考文章

http://www.cnblogs.com/yydcdut/p/3959815.html

https://segmentfault.com/q/1010000006669618

https://www.javadoop.com/post/hashmap#Java7%20ConcurrentHashMap