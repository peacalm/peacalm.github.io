---
title: "What Is a Good Local Cache"
date:    2022-03-31T02:15:21+08:00
lastmod: 2022-03-31T02:15:21+08:00
draft: false
tags: ["缓存", "Cache", "LocalCache", "Wrapper", "SideCar"]
categories: ["技术"]
---

缓存技术，Cache，特别是LocalCache，是软件开发中非常常用的组件，也是提高性能的最简单的方式。

# Cache的两种使用模式：SideCar模式和Wrapper模式

SideCar模式：需要应用程序自己主动访问数据源服务并读取缓存。  

这种模式可以是LocalCache，与应用程序集成部署，但是由于Cache功能与业务逻辑无耦合，非常独立，所以出现了很多独立出来的优秀的第三方缓存服务，
比如Redis，Memcached，cachelib等。
独立缓存服务与应用程序分开，独立部署，需要通过网络进行交互，数据量较大或对性能要求较高时，网络带宽消耗、延时、数据序列化反序列化的CPU消耗不能忽视。
```text
Cache
↑  
|  
App  -->   DataSource
```

Wrapper模式：应用程序只需要访问缓存，缓存具有代理能力可以帮助应用程序去访问数据源服务。  

这种模式需要对Cache做特殊化定制，一般是LocalCache，与应用程序集成部署，只需在内存中进行交互，性能较高。如果Cache设计的好，可以极大简化业务代码，
可以提高组件可复用性。
```text
App --> Cache --> DataSource
```

那么怎么才算是一个好的LocalCache呢？  

# 灵活好用

* 可以动态修改CacheSize，一般指缓存的数据条目数。
* 具有单Key读写接口，也具有Batch读写接口。
* 可以设置一个数据的默认过期时间，也可以动态设置每一个Key的过期时间，特别的，在插入或读取时都可以设置和判断过期。
* 可以配置读取数据源服务的CallbackOnFetching方法，以便应用Wrapper模式，当Key不存在或过期时，用来获取新数据。
* 过期数据不自动删除，可配置一个CallbackOnExpiration方法来注入对过期数据的处理方式。比如可以删除数据，或把过期数据发送到某个消息队列、写入硬盘等，
或不对过期数据做任何处理，只等缓存用满以后自动逐出。
* 支持多种QueryFlags:
  * uncacheable: 本次Query不读写Cache，只从数据源服务获取数据并返回，不写入Cache，相当于没有Cache。
  * refresh:     本次Query不读取Cache数据，重新从数据源服务获取最新数据，并写入Cache，相当于刷新缓存。
  * peek:        本次Query只读取Cache数据，即时Miss或Expired也不从数据源服务获取最新数据。
  * stale:       本次Query读取Cache数据时，可以接受过期的数据。
  * fast:        本次Query只返回Cache里的数据，如果Miss或Expired，则发送reload任务刷新缓存，不阻塞当前Query。
  * preload:     本次Query如果在Cache中读取到合法数据，但是数据快过期了，比如已过了过期时间的80%，则发送一个eload任务来刷新缓存。
* 内置线程池可执行Reload任务，从数据源服务获取数据并填充Cache。
* 可配置内置WatchDog现成，检查快过期的数据，并发送Reload任务。
* 可配置缓存策略，例如：LRU，LRU-k(lazy list adjustment)，FIFO等。
* 可迭代操作Cache中的每一条数据，例如把Cache中的数据读出写入到其他设备。

# 高性能
* 并发性好。例如采用分桶机制、采用TBB的高性能并发容器库等。
* 内存使用效率高。

