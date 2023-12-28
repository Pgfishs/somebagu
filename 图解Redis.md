# Redis
### 什么是redis
基于内存的数据库，对数据的读写都是在内存中完成的，读写速度非常快，用于缓存、消息队列、分布式锁登场景
redis提供了多种数据类型，String、Hash、List、Set、Zset、Bitmaps、HyperLogLog、GEO、Stream，对数据类型的操作都是原子性的，因为**执行命令由单线程负责**，不存在并发竞争的问题
redis还支持事务、持久化、lua脚本、多种集群方案、发布/订阅模式、内存淘汰机制、过期删除机制等