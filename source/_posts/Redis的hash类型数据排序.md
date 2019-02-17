---
title: Redis的hash类型数据排序
date: 2019-02-17 19:00:48
tags: 
  - redis
  - 数据库
---
最近在研究如何使用redis来做数据的缓存服务，遇到一个hash数据排序的问题，感觉网上资料也是乱糟糟的，自己整理如下。

对于hash类型数据的排序，需要两个部分。

1. 多个hash类型的数据（废话）
2. 保存hash数据ID的列表。

使用sort命令，利用by hash数据的field的内容来对ID的列表进行排序。

我们首先插入点hash类型的试验数据。
```
127.0.0.1:6379> hset student:1 name tom score 85
(integer) 2
127.0.0.1:6379> hset student:2 name john score 60
(integer) 2
127.0.0.1:6379> hset student:3 name bob score 75
(integer) 2
127.0.0.1:6379> hset student:4 name mike score 95
(integer) 2
```
一共插入4个hash数据。下面我们建立跟这4个数据对应的ID列表
```
127.0.0.1:6379> lpush id 1
(integer) 1
127.0.0.1:6379> lpush id 2
(integer) 2
127.0.0.1:6379> lpush id 3
(integer) 3
127.0.0.1:6379> lpush id 4
(integer) 4
```
下面我们祭出sort大法，来根据score进行排序。
```
127.0.0.1:6379> sort id by student:*->score get student:*->name
1) "john"
2) "bob"
3) "tom"
4) "mike"
```
可以看到数据是按照从小到大的顺序排序的。如果想从大到小，加上desc就好。
```
127.0.0.1:6379> sort id by student:*->score get student:*->name desc
1) "mike"
2) "tom"
3) "bob"
4) "john"
```




