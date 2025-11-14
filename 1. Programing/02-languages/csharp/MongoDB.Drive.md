# 使用
---
### 準備
* nuget 包
```
MongoDB.Driver
```

## Connection
* Connection String
```cs
const string connStr = "mongodb://localhost:27017";

var client = new MongoClient(connStr);
```
> `MongoClient` 本身就是數據庫連接池，所以就算有多個 Request 也用通一個實例就行。

* 其他 Connection Options: 請閱讀 [Documentation](https://www.mongodb.com/docs/drivers/csharp/current/fundamentals/connection/connection-options/)

## Database and Collections
### 簡介
MongoDB 使用垂直方式來管理數據，一個 MongoDB 管理多個 **database**，而 database 又管理多個 **collections**，每一個 collection 中數據使用 **document** 存放。
> database > collection > document

### Database 操作
```cs
// access or create
var db = client.GetDatabase("db_name"); // 若不存在，在後續插入數據時候會建立

// list
await client.ListDatabaseNamesAsync();

// drop
await client.DropDatabaseAsync("db_name");
```
### Collection 操作
```cs
// access or create
var cols = client.GetCollection<BsonDocument>("coll_name"); // 若不存在，在後續插入數據時候會建立

// list
await client.ListCollectionNamesAsync();

// drop
await client.DropCollectionAsync("coll_name");
```


## CRUD Operations

### Write
```cs
// insert

// modify

// delete
```
### Read







