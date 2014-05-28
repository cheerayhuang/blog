Tags: mysql 数据库
#用可扩展的方式使用mysql

读了老外写的一些文章，记录一下。
原文在这里： 
http://www.slideshare.net/billkarwin/extensible-data-modeling#
http://backchannel.org/blog/friendfeed-schemaless-mysql

##0 当要为关系数据库表增加属性时，会发生什么
> 1.  Lock the table. 
2.  Make a new, empty the table like the original. 
3.  Modify the columns of the new empty table. 
4.  Copy all rows of data from original to new table… 
    no matter how long it takes. 
5.  Swap the old and new tables. 
6.  Unlock the tables & drop the original.

对于msyql 这样的关系数据库来说， 首先它会将这张表锁住，然后建立一张完全一样的空表。接着根据用户的要求修改这张表的属性名（列名），并且逐条把记录从原来的表拷贝到新的表。然后用新表替代原来的表， 解锁用来的表。

##1 预先创建多个列

这个方法是一种很直接的方法，在创建一张mysql表的时候，我们可以提前创建一些**预留**的列，比如这样：

```SQL
CREATE TABLE Title (
 id int(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
 title text NOT NULL,
 imdb_index varchar(12) DEFAULT NULL,!
 kind_id int(11) NOT NULL,
 production_year int(11) DEFAULT NULL,!
 imdb_id int(11) DEFAULT NULL,
 phonetic_code varchar(5) DEFAULT NULL,
 extra_data1 TEXT DEFAULT NULL,
 extra_data2 TEXT DEFAULT NULL,
 extra_data3 TEXT DEFAULT NULL,
 extra_data4 TEXT DEFAULT NULL,
 extra_data5 TEXT DEFAULT NULL,
 extra_data6 TEXT DEFAULT NULL,
);

```

上面SQL语句中的9-14列，都是**预留**的属性，当项目有额外的属性需求的时候，就直接利用这些列就好了。

这个解决方案的优点是： 
> No ALTER TABLE necessary to use a column for a new 
attribute—only a project decision is needed.

缺点是： 
> * If you run out of extra columns, then you’re back to 
ALTER TABLE. 
> * Anyone can put any data in the columns—you can’t 
assume consistent usage on every row. 
> * Columns lack descriptive names or the right data type.


##2 创建横向的属性表

这个解决方案可以创建一张这样的属性表，如下： 

```SQL
CREATE TABLE Attributes ( 
 entity INT NOT NULL, 
 attribute VARCHAR(20) NOT NULL, 
 value TEXT, 
 FOREIGN KEY (entity)  
 REFERENCES Title (id) 
);
```

这个表的第二列是属性名字，这样你可以自由的扩展属性了，并把属性对应的值填在第三列，像这样：

> 
+--------+-----------------+---------------------+
| entity | attribute | value |
+--------+-----------------+---------------------+
| 207468 | title     | Goldfinger |
| 207468 | production_year | 1964 |
| 207468 | rating | 7.8 |
| 207468 | length | 110 min |
+--------+-----------------+---------------------+

但是你为了得到一个完整的对象的查找结果，无疑会很麻烦。你必须得这样： 

```SQL
SELECT a.entity AS id,
 a.value AS title,
 y.value AS production_year,
 r.value AS rating,  b.value AS budget!
FROM Attributes AS a
JOIN Attributes AS y USING (entity)
JOIN Attributes AS r USING (entity)
JOIN Attributes AS b USING (entity)
WHERE a.attribute = 'title'
 AND y.attribute = 'production_year'
 AND r.attribute = 'rating'  AND b.attribute = 'budget';
```
属性在这里已经成为一列值了，所以在查询的时候需要大量的where语句对结果进行约束。

这个解决方案的好处是： 
> * No ALTER TABLE needed again—ever! 
> * Supports ultimate flexibility, potentially any “row” can 
have its own distinct set of attributes. 

坏处也是显而易见的：
> * SQL operations become more complex. 
> * Lots of application code required to reinvent features 
that an RDBMS already provides. 
> * Doesn’t scale well—pivots required. 

##3 利用继承的方式扩展

表的继承主要是利用外键的关联来实现。这样一来，你可以根据需求的扩展，创建新的表然后与原来的表用外键关联起来（一般来说，通过主键，比如ID）。这样也实现了表的动态扩展。 

比如下面这个例子，根据电影和电视节目的属性，增加了新的需求，于是额外扩展了两张子表： 

```SQL
CREATE TABLE Title (
 id int(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
 title text NOT NULL,
 imdb_index varchar(12) DEFAULT NULL,
 kind_id int(11) NOT NULL,
 production_year int(11) DEFAULT NULL,
 imdb_id int(11) DEFAULT NULL,
 phonetic_code varchar(5) DEFAULT NULL,
 title_crc32 int(10) unsigned DEFAULT NULL,
 PRIMARY KEY (id)
);

CREATE TABLE Film (!
 id int(11) NOT NULL PRIMARY KEY,  aspect_ratio varchar(20),
 FOREIGN KEY (id) REFERENCES Title(id)
);

CREATE TABLE TVShow (
 id int(11) NOT NULL PRIMARY KEY,
 episode_of_id int(11) DEFAULT NULL,
 season_nr int(11) DEFAULT NULL,
 episode_nr int(11) DEFAULT NULL,
 series_years varchar(49) DEFAULT NULL,
 FOREIGN KEY (id) REFERENCES Title(id)!
);

```
![表继承][1]

这时候比如来了一个新的需求，videoGames， 你的表需要多加几个属性，你可以这样:

```SQL
CREATE TABLE VideoGames ( 
 id int(11) NOT NULL PRIMARY KEY, 
 platforms varchar(100) NOT NULL, 
 FOREIGN KEY (id)  
 REFERENCES Title(id) 
);
```

把videoGames的属性建一张新表，不需要改原来的表，然后通过外键的方式与原来的表关联起来，实现继承。

这个解决方案的好处：
> * Best to support a finite set of subtypes, which are likely 
unchanging after creation. 
> * Data types and constraints work normally. 
> * Easy to create or drop subtype tables. 
> * Easy to query attributes common to all subtypes. 
> * Subtype tables are shorter, indexes are smaller.

坏处是： 
> * Adding one entry takes two INSERT statements. 
> * Querying attributes of subtypes requires a join. 
> * Querying all types with subtype attributes requires 
multiple joins (as many as subtypes). 
> * Adding a common attribute locks a large table. 
> * Adding an attribute to a populated subtype locks a 
smaller table.

##4 利用序列化直接存储对象

这种方式很直白，就是直接把对象的属性序列化成一定的格式，作为字符串之后直接存储在表里。

```SQL
CREATE TABLE entities (
    added_id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    id BINARY(16) NOT NULL,
    updated TIMESTAMP NOT NULL,
    body MEDIUMBLOB,
    UNIQUE KEY (id),
    KEY (updated)
) ENGINE=InnoDB;
```
其中body的内容是一个很长的序列化后的字符串，比如json格式的：

```Json
{
    "id": "71f0c4d2291844cca2df6f486e96e37c",
    "user_id": "f48b0440ca0c4f66991c4d5f6a078eaf",
    "feed_id": "f48b0440ca0c4f66991c4d5f6a078eaf",
    "title": "We just launched a new backend system for FriendFeed!",
    "link": "http://friendfeed.com/e/71f0c4d2-2918-44cc-a2df-6f486e96e37c",
    "published": 1235697046,
    "updated": 1235697046,
}
``` 

显而易见的是，这样一来，完全不存在根据需求扩展属性的问题了。但是查询的时候因为所有的属性都已经被序列化在一起，所以会带来麻烦。 

如果是序列化成xml格式的话，mysql提供了一些简单的支持, 比如： 

```SQL
SELECT id, title,  
 ExtractValue(extra_info, '/episode_nr')  
 AS episode_nr  
FROM Title 
WHERE ExtractValue(extra_info,  
 '/episode_of_id') = 1292057;
```
可以在sql语句里直接解开xml属性

而与mysql同源的MariaDB，则对利用json创建动态列，提供了一些支持：
```SQL
INSERT INTO Title (title, extra_info)  
VALUES ('Trials and Tribble-ations',  
 COLUMN_CREATE('episode_of_id', '1291895',  
 'episode_nr', '5', 
 'season_nr', '6'));
```
直接把json写入列中。

显然这些数据库对于这种方式的支持都是有限的。可以利用自己维护一个索引，来解决。

这个解决方案的优点是：
> * Store any object and add new custom fields at any time. 
> * No need to do ALTER TABLE to add custom fields

缺点是： 
> * Not indexable. 
> * Must return the whole object, not an individual field. 
> * Must write the whole object to update a single field. 
> * Hard to use a custom field in a WHERE clause, GROUP BY 
or ORDER BY. 
> * No support in the database for data types or 
constraints, e.g. NOT NULL, UNIQUE, FOREIGN KEY

##5 在序列化存储的方式上，增加额外的反向索引

在4方案的基础上，增加一个反向索引，还是用上面4中的例子。

```SQL
CREATE TABLE index_user_id (
    user_id BINARY(16) NOT NULL,
    entity_id BINARY(16) NOT NULL UNIQUE,
    PRIMARY KEY (user_id, entity_id)
) ENGINE=InnoDB;
```
给body的json格式里的entity_id 这个key增加了一个index表，那么利用这个反向索引表，我们可以先找到id，然后再去主表中根据id查到我们想要的对象。如果有其他属性需要索引，我们可以建立额外的索引表，就像上面那样，让一个属性与id相关联就可以了。

对于写入一个新的对象，我们把对象添加进主表之后，还要修改所有的索引表，总的来说写入的过程是这样的：
> 1. Write the entity to the entities table, using the ACID properties of InnoDB
> 2. Write the indexes to all of the index tables on all of the shards

这个过程，第一步可以利用Transaction（ACID)保持原子操作。 但是整个过程，第一步与第二步，并非原子操作。

于是，我们查询一个对象的时候，需要加入一个过滤结果的过程，来应对反向索引不准确的情况（当写入操作的第二步没有被执行的时候，索引并没有正确更新）。读操作的过程如下：
> 1. Read the entity_id from all of the index tables based on the query
> 2. Read the entities from the entities table from the given entity IDs
> 3. Filter (in Python) all of the entities that do not match the query conditions based on the actual property values

当然，还需要一个后台运行的进程来修正错误的反向索引，以及清除失效的索引。这个后台运行的进程，应该总是有先清理那些近期更新的反向索引。


  [1]: ./res/extendible_mysql_model_fig1.png