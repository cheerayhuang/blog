# 游戏服务器存储过程api设计和讨论

标签（空格分隔）： 游戏服务器 数据库 redis mysql

---

[TOC]

<!--[TOC] -->

##1 存储过程的讨论

###1.1 采用redis作为cache, mysql作为database的方案

####1.1.1 mysql是否需要分表

---

####1.1.2 读操作
1. 当读操作发生时，首先判断数据是否在redis中缓存，如果是，则直接读取返回
2. 如果没有在redis中缓存，则把读操作落到mysql上，从mysql中读取
3. 将读到的对象数据更新至redis缓存，返回。

简单的时序图如下所示

![read_obj](http://www.websequencediagrams.com/cgi-bin/cdraw?lz=I-WuouaIt-err-S4juacjeWKoeWZqOerr-WvueixoeaVsOaNruivu-aTjeS9nOeugOWNleaXtuW6jwoKI-iiq-WKqOWkseaViOetlueVpQoKY2xpZW50LT5zZXJ2ZXI6IFthcGld6K-7AEoGCgoAEgYtPnJlZGlzABgH5Yik5patAGYM5piv5ZCm5ZyoACIF5LitCgphbHQgAIEJDAARDCAgICAARRTku44AOQjor7vlj5YAgUoMCmVsc2UARg3kuI0APhhteXNxbABMCgAKBQBAFgB0GOWwhuivpQCBWAeUvuWFpQCBVAjnvJPlrZgKZW5kCg&s=rose)

> * 这一读操作方案缺点是， 从数据在redis中缓存失效直到重新把数据缓存在redis中，这段时间内，所有的读操做都会落在mysql上，加大了mysql的负担。
> * 可以用写操作去做弥补，当数据库更新时，主动去同步更新缓存，这样在缓存数据的整个生命期内，就不会有空窗期，前端请求也就没有机会去亲密接触数据库。
> * 也可以设置一个flag， 只保证最终只有一个请求去直接读mysql数据库，并更新redis， 其他请求返回错误，让客户端再试，那么下次读redis的时候就能够命中这个对象了。这样的做法增加了读操作的时间开销。

---

####1.1.3 写操作，方案1
1. 当写操作发生时， 直接写redis并缓存
2. 将redis缓存每过一定的时间间隔，同步写到mysql中。

简单的时序图如下图所示
![write_obj_redis](http://www.websequencediagrams.com/cgi-bin/cdraw?lz=I-WGmeaTjeS9nO-8jOaWueahiDHvvIznm7TmjqXlhplyZWRpcwoKY2xpZW50LT5zZXJ2ZXI6IFthcGld5YaZ5a-56LGhCgARBi0-ACkFABYInKgAOAXkuK3lop7liqDmiJbogIXkv67mlLnor6UANAdsb29wICDmr4_pmpTkuIDkuKrml7bpl7Tpl7TpmpQKICAgIABWCG15c3FsAHQIsIYAgRYFAHsG5ZCM5q2l5YiwAB4F5LitCmVuZAo&s=rose)

> * 由于是直接先写的redis， 所以解决了读操作的空窗期的问题（当然数据超时了失效，这种情况除外）。
> * 但是这样的写操作会带来redis并发性的问题, 因为redis底层还是单线程写操作的。当多个请求并发读取同一个数据并修改的话，后写的数据会覆盖前面写的数据，导致数据丢失。这个问题是否需要解决可以看该数据的使用场景。如果解决，可以使用加锁（不推荐）或者 数据版本控制的方式解决这个问题，给数据带上版本号，在写入前加一次版本号检查，如果已经存在的版本号比当前的版本号高，那么就不写入。

---

####1.1.4 写操作，方案2
1. 当写操作发生时，直接写mysql数据库
2. 并且将更新的数据对象，缓存到redis中

简单的时序图如下图所示
![write_obj_mysql](http://www.websequencediagrams.com/cgi-bin/cdraw?lz=I-WGmeaTjeS9nO-8jOaWueahiDLvvIzlhYjlhplteXNxbAoKY2xpZW50LT5zZXJ2ZXI6IFthcGldIOWGmeWvueixoQoAEgYtPgAqBQAWCbCGABkG5pWw5o2u5YaZ5YWlAEsGACgIcmVkaXMAHxEAJAYAFwXnvJPlrZgKCg&s=rose)

> * 这个方案实现简单，没有同步过程。但是直接只把redis当作读缓存，在写操作上，redis没有作用。
> * 也从写操作上，解决了读操作时的缓存空窗期问题。
> * Mysql在正常使用下可以达到1200左右的tps(来自mysql官网数据，4-8线程并发读写）。对于小规模的手机游戏来说，这个方案虽然没有写缓存，但依然可以考虑。
> * 相比第一个写方案， 这个方案依然存在着并发写数据丢失的问题，只不过这个并发问题落在了mysql上，而不是redis上。

---

###1.2 采用redis作为database, mysql或者更轻量级的Unqlite作为数据落地备份的方案

* 这个方案直接使用redis作为database, 服务器启动的时候将数据从mysql（或者其他数据库，下同）导入redis。
* mysql只是用来实现数据持久化，以及数据的备份
* 当有对象在redis中发生更新的时候，只需要在mysql中再写一份就行了。
* 对于长期不用的对象，可以在redis中清理掉，当再需要用的时候（比如很久不登录的玩家重新登录），只需要从mysql中恢复回来即可。

---

##2 API设计的讨论

###2.1 API返回结果

####2.1.1 错误结果

错误的结果采用以下的格式, 暂定为json协议

    {
        error -- 错误代码，没有错误时为0 
        err_msg -- 错误描述，没有错误时为空字符串
    }

---

####2.1.2 正确的结果

直接返回对象的内容，暂定为json协议

---

###2.2 API接口

####2.2.1 根据对象名获得id
    get_uid(name) 

* 在mysql中，使用sql语句执行select
* 在redis中缓存，id与name为一个键值对，应该是 表名:id  ---> name的形式： 
`set "login_table:3"  "hqy"` 

---

####2.2.2 查找所有对象
    find_all_obj([table_name]) 

* table_name可以由服务器端提供
* 在mysql中，通过执行select语句查询。如果是采用多表存储对象，那么应该使用transaction，将多个表的查询结果组合成对象返回
* 在redis中，通过形如"表名: *" 的键形式来查询, 然后组合结果成为对象返回

---

####2.2.3 根据id查找对象
    find_by_id([table_name], id) 
     
* table_name 可以由服务器端提供。
* 在mysql中， 通过执行select 语句查询。如果是采用多表存储对象，那么应该使用transaction，将多个表的查询结果组合成对象返回
* 在redis中，通过形如"表名:id:porperty_name" 的键形式来查询出所有对象的属性结果, 然后组合结果成为对象返回

---

####2.2.4 根据某一属性的范围来查找对象

    find_by_condtion([table_name], condition_judgement_func) 

* table_name 可以由服务器端提供。
* 在mysql中， 使用条件子句，进行select查询。如果是采用多表存储对象，那么应该使用transaction，将多个表的查询结果组合成对象返回
* 在redis中，使用sorted set 以及list 等数据结构，进行额外的存储，才能完成查询。

---

####2.2.5 根据某一属性的降序或升序返回对象

    find_by_property_ascend([table_name], property_name) 
    
    find_by_property_descend([table_name], property_name)
    
* table_name 可以由服务器端提供。
* 在mysql中， 使用条件子句 "order by“，进行select查询。如果是采用多表存储对象，那么应该使用transaction，将多个表的查询结果组合成对象返回
* 在redis中，使用sorted set数据结构，进行额外的存储，并查询。

---

####2.2.6 对于多表的联合查询以及其他复杂查询

    find_by_union_condition([table_name], ..., property_name ...) 

* 在mysql中，使用联合查询，以及其他复杂的sql语句形式（比如嵌套）
* 在redis中，需要提前根据需求设计好映射关系，才能进行查询

---

###2.2.7 写入对象

    write_obj([table_name], object) 

* object中并不包含 "id"， 这一属性由 redis 和 mysql产生
* 在mysql中，将一个完整的对象结构写入数据库，如果是多表，需要写入多次，可以使用transaction
* 在redis中， 将这一个对象的多个属性，插入多个键值对集合中

---

###2.2.8 更新对象的一个或者多个属性（可以与2.2.7的接口合并）

    write_obj_property([table_name], property_name1, ...) 
    
* 在mysql中，使用update 更新一个对象的多个属性，如果是多表，需要写入多次，可以使用transaction
* 在redis中，更新多个键值对集合



