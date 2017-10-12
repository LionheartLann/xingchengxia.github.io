---
title: MongoDB notes
date: 2017-08-22 11:58:49
tags:
---
# MongoDB: The Definitive Guide notes
#lannister/notes
## Introduction
* document-oriented,  不采用关系型为了更好的扩展性
* 易于横向扩展，在多台机器间分割，自动处理跨集群的数据和负载
* 索引(indexing)：二级索引／多种快速查询／唯一索引／复合索引／地理空间索引／全文索引
* 聚合(aggregation)： aggregation pipeline  
* 特殊集合类型：时间有限／空间有限
* file storage
* 不具备join／事务
* dynamic padding 额外空间换取稳定性能
* 将处理和逻辑交给client

## Fundamental
### Document && Collection && db
* _id unique, 相当于一行
* 一个键值对的有序集，区分类型、大小写
* 不能有重复的键
* collection相当于一张表
* 保留数据库：admin用户权限/ local /config 分片
启动：`mongo` ，同时启动一个http服务器`port 28017`
* 数据类型
null／布尔 ／ 数值（64位 float）NumberInt(“3”) NumberLong(“3”)／字符串／日期 new Date() 毫秒数／数组／RE _foobar_i ／内嵌文档／对象ID ObjectId() ／二进制／JS代码
* _id组成: 时间戳0123|机器456|PID 78 |计数器 9 10 11

### Shell
完备的JS解释器
* CRUD
`use dbname`
`db.insert(post)`
`db.find()` `db.findOne()`
`db.update({title:”My blog post”}, post)`
`db.remove({title:”My blog post”})`
* `mongo some-host:30000/myDB` 连接指定db
启动时不连接: `mongo —nodb`，启动后
```
conn = new Mongo(some-host:30000)
db = conn.getDB('myDB')
```
* 执行JS：`mongo *.js` `load(“your.js”)`
* `.mongorc.js` －－启动时自动加载
可用于创建全局变量、移除较危险的操作（防误操作）
 great feature: `EDITOR=“/usr/local/bin/vim”`，可放入`~/.mongorc.js`。调用`edit something` 即可
`db.dropDatabase = Db.prototype.dropDatabase=no;`
`DBCollection.prototype.drop=no;`
`DBCollection.prototype.dropIndex=no;`
* 集合命名
获取集合：`db.getCollection(“version”)`
子集合 `db.name <==> db[“name”]`


## CRUD

### 插入
* 批量插入 `batchInsert` ，同`Insert()`，接受数组变量
* 单个文档 size < 16M
* 校验
### 删除文档
`db.foo.remove()` 删除所有文档，`db.foo.drop()`删除集合（fast）
### 更新文档
使用`_id`指定文档，避免出现重复
* 修改器（update modifier）
原子性
```
$inc
$set/ $unset
$push
$each
$slice
$sort
$addToSet
$pop $pull
```
upsert
getLastError
findAndModify

## 查询
查询name为xx，age为27的人：
`db.users.find({"name":"xx", "age":27})`
select 返回需要的列, find的第二个参数指定：
`db.users.find({},{"gender":1,"addr":0})`
### 查询条件
```
$lt $lte $gt $gte $ne 
$in $nin 类似mysql in {"something":{"$in":[]}}
$or 多个键 {"$or":[{},{}]}
$mod 取模运算 
$not 常与re联合使用
$exists null
```
### regular expression
模糊匹配
`/rtext/i`
＊前缀re可创建索引

### 查询数组
查询指定位置元素
查询指定长度数组
返回一个匹配的数组元素
数组的范围查询
### 查询内嵌文档
### $where
借助JavaScript 完成 
不能使用索引
把每个文档转换为BSON对象，慢
### 服务端脚步
防注入／作用域 传递
### 游标
### limit、skip、sort
tips：减少扫描行数
一个思想：返回集合中的一个随机文档
### wrapped query 封装查询
### runCommand：接受一个命令文档：
`db.runCommand({getLastError:1})`

## Indexing
首先建立测试数据库：
`for (i=0; i<10000000; i++) { db.users.insert({"i":i, "name":"tk"+i, "age":Math.floor(Math.random()*120), "created":new Date()}); }`
### 索引类型
* 唯一索引：确保Doc指定键有唯一值，如`_id`.
如果一个Doc没有指定键，将作为`null`存储。如果在其上建立唯一索引，但又插入多个缺少该键的Doc，将导致插入失败。
* 复合唯一索引：组合值唯一
* 复合索引(compound index)
`db.users.ensureIndex({”age”:1,“name”:1})`
索引的使用方式取决于查询的类型：
* point query
`db.users.find({"age":21}).sort({"name":-1})`
非常高效：直接定位年龄，不需要对结果排序（只需对数据进行逆序遍历即可得到正确的顺序）
* multi-value query
`db.users.find({"age":{"$gte":21, "$lte":30}})`
使用索引的第一个键匹配Doc

`db.users.find({"age":{"$gte":21, "$lte":30}}).sort({"name":1})`
结果集中name是无序的，需要在内存中排序，不如上一个高效
### .explain() 相当于MySQL explain
`db.users.find({}).explain`
- [x] 结果不一样？没有mills等key
Excute with out index:
```
db.users.find({"name" : "tk101"}).explain('executionStats')

"executionStats" : {
		"executionSuccess" : true,
		"nReturned" : 1,
		"executionTimeMillis" : 5145,
		"totalKeysExamined" : 0,
		"totalDocsExamined" : 10000000,
		"executionStages" : {
			"stage" : "COLLSCAN",
			"filter" : {
				"name" : {
					"$eq" : "tk101"
				}
			},
			"nReturned" : 1,
			"executionTimeMillisEstimate" : 4709,
			"works" : 10000002,
			"advanced" : 1,
			"needTime" : 10000000,
			"needYield" : 0,
			"saveState" : 78156,
			"restoreState" : 78156,
			"isEOF" : 1,
			"invalidates" : 0,
			"direction" : "forward",
			"docsExamined" : 10000000
		}
	}
```
See Docs: [cursor.explain](https://docs.mongodb.com/manual/reference/method/cursor.explain/) 

* `{“sortKey" : 1, "queryCriteria" : 1}`
### 加索引
`db.users.ensureIndex({“name”:1})`
加复合索引
使用索引排序非常快，但是必须首先使用索引
`db.users.find().sort({"age":1,"name":1})`
`db.users.ensureIndex({”age”:1,“name”:1})`



