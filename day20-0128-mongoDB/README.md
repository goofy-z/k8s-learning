### 安装MongoDB





### DB使用

1. 连接mongo

   - 直接输入mongo的db 验证用户和密码

     mongo --host XXXXXX --port XXXXXX --authenticationDatabase 数据哭 -u 数据库用户 -p 数据库密码

   - 先进入mongo shell，然后use db 再来验证

     mongo --host XXXXXX --port XXXXXX 进入客户端shell，通过use切换到目标db，然后通过db.auth("xxxxxxx","xxxxxx") 进入客户端后进行认证

   mongo => use admin => db.auth('admin', '123456')

2. 创建、删除数据库，表

   - 查看当前数据库  `db`;
   - 查看所有数据库 `show dbs`; 
   - 查看说有集合（表）`show tables` or `show collections`
   - 创建数据库 `use xxx`
   - 创建集合 db.createCollection("test_table")

   - 删除当前数据库: `db.dropDatabase() `  只能删除当前数据库
   - 删除集合  `db.collection.drop("test_collection")`

3. 增删改查

   - 插入文档 `db.test_table.insert({title: "abcd"})` 表不存在就自动建表

   - 查看文档  `db.test_table..find()`

     ```json
     { "_id" : ObjectId("5e4670584825eaa610390f7c"), "name" : 123 }
     ```

     其中ObjectId 为按时间戳机器等生成的随机id 前8位是16进制的ISO时间戳也就是标准时间

     解析日期 `ObjectId("5e4670584825eaa610390f7c").getTimestamp()`

   - 