## MongoDB

#### 数据库

1. 关系型数据库(RDBMS)：mysql , oracle, db2, sql server
   * 关系数据库中全是表
2. 非关系型数据库(NoSQL)：MongoDB、redis
   * 键值对数据库(redis)
   * 文档数据库(MongoDB)

******

#### 简介

* MongoDB的数据模型是面向文档的，所谓文档就是类似于JSON的结构，MongoDB中存的就是各式各样的JSON(BSON)。

******

#### 安装

![mongodb安装](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/mongodb1.png)

* 可以配置开机自启动，菜鸟教程

******

#### 基本概念

* 数据库(database)
* 集合(collection) == 表
* 文档(document) == 记录(元组)
  * 数据库>集合> 文档
  * 在MongoDB中，数据库和集合都不需要手动创建
  * 当我们创建文档时，如果文档所在的集合或数据库不存在，会自动创建

*******

#### 基本指令

* show dbs  == show databases
* use 数据库名
* db : 显示当前所在的数据库
* show collections : 显示数据库中的所有集合

**CRUD操作**

* 插入文档

  * db.集合名称.insert(文档对象)

    * 向集合中插入一个或多个文档

    * 当像集合中插入文档时没有给文档指定_id，数据库会自动生成一个，该属性用来做唯一标识，_id 也可以自己指定但要唯一。

    * 例子：向test数据库中的stus集合插入一个新的学生对象

      {name:"zhangsan",age:10,gender:"male"}
    
      db.stus.insert({name:"zhangsan",age:10,gender:"male"})
      
    * 插入多个对象
    
      ```shel
      db.stus.insert([
      	{name:"lisi", age:"12", gender:"female"},
      	{name:"王五", age:"15", gender:"male"}])
      ```
      
    
  * db.集合名称.insertOne(文档对象)
  
    * 只能插入一个文档对象。
  * db.集合名称.insertMany(文档对象)
    * 只能插入多个文档对象。
    * 这样做更清晰。
  
* 查询文档
  * db.集合名称.find()
    
    * 查询当前集合的所有文档
    
    * 可以接受一个对象作为条件参数
    
      * {}: 表示查询集合中所有文档
      * {属性:值}: 查询指定属性值的文档
    
    * find() 返回的是一个数组
    
      ```she
      db.stus.find({name:"lisi"})
      ```
  
  * db.集合名称.findOne()
  
    * 查询集合中符合条件的第一个文档
  
    * findOne() 返回的时一个文档对象
  
      ```she
      db.stus.findOne({name:"zhangsan"})
      ```
  
  * db.集合名称.find({}).count()
    
  * 查询所有结果的数量
  
* 修改文档

  * db.集合名称.update(查询条件,新对象,属性`可以没有` )
    * update() 默认情况下使用新对象替换旧对象
    * 如果需要修改指定的属性，而不是替换需要使用"修改操作符"来完成
      * $set 可以用来修改文档中的指定属性
      * $unset 可以用来删除文档的指定属性
    * update() 默认只会修改一个
    * 属性可以指定multi为true，这样就可以修改多个对象

  * db.集合名称.updateMany()

    * 同时修改多个符合条件的文档

  * db.collection.updateOne()

    * 修改一个符合条件的文档

      ```shell
      // 修改所有name为lisi的年龄为20
      db.stus.updateMany(
      	{name:"lisi"},  // 查询条件
      	{				// 新对象
      	    $set:{
      	        age:20
      	    }
      	}
      )
      ```

  * db.集合名称.replaceOne()

    * 替换
  
* 删除文档

  * db.集合名称.remove()
    * 删除一个或多个，可以传第二个参数，若为true则只会删除一个
    * 如果传一个空对象则会删除全部
  * db.集合名称.deleteOne()
  * db.集合名称.deleteMany()
  * db.集合名称.drop() 删除集合
  * db.dropDatabase() 删除数据库

**文档间的关系**

1. 一对一
2. 一对多 多对一
3. 多对多

可以通过内嵌文档的形式来体现

**排序和投影**

* db.collection.find({}).sort({})

  *sort()需要传递一个对象来指定排序规则 1表示升序，-1表示降序*

* db.collection.find({})

  find() 可以在第二个参数的位置来设置查询结果的 投影 (想要显示的列) 想显示的 1 不要显示的 0 

#### 用户权限

开启权限验证...

mongodb可以给每个数据库都指定一个用户管理，admin数据库则是管理这些用户。

连接mongo

```powershell
./bin/mongo
```

* **查看当前库下的所有用户**

```powershell
> use admin
> db.auth("root","123456")
> show users  // 查看当前库下所有用户(admin)
> db.system.users.find()  // 查看数据库所有用户
```

* **新建用户**

```powershell
use test // 切换到指定数据库
db.createUser(
	{
		user:"user",
		pwd:"123456",
		roles:[{
			role:"readWrite",
			db:"test"
		}]
	}
)
```

* **删除用户**

```powershell
db.dropUser("user")
```

* **更新用户**

```powershell
db.updateUser("user",{
	roles:[
		{role:"readWrite",db:"test"},
		{role:"dbAdmin",db:"test"}
	]
})
```

* **更改密码**

```powershell
db.changeUserPassword("user", "密码")
```

* **授予权限**

```powershell
db.grantRolesToUser("user", [{role:"uesrAdmin", db:"test"}])
```

* **撤销权限**

```powershell
db.revokeRolesFromUser("user",[{role:"userAdmin", db:"test"}])
```

* **权限角色**

Read：允许用户读取指定数据库

readWrite：允许用户读写指定数据库

dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile

userAdmin：允许用户向system.users集合写入，可以找指定数据库里创建、删除和管理用户

clusterAdmin：只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限。

readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限

readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限

userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限

dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限。

root：只在admin数据库中可用。超级账号，超级权限

* 关于认证机制问题

mongodb数据库低版本使用的认证机制是`MONGODB-CR`，较高版本则是已经换成`SCRAM-SHA-1`更加安全 。但是在使用springboot集成mongo时，较低版本的例如1.5.x采用的默认的`MONGODB-CR`认证机制，所以出现了认证异常。解决：升级springboot版本、修改mongodb数据的认证方式/降低mongodb版本、将连接uri中指定认证机制版本。

1. 升级springboot版本，在一个系统架构都已经确定的情况下就不推荐了

2. 修改mongodb数据的认证方式，然而我用的mongodb版本为`4.0.19`，mongodb数据库4.0以上的版本已经不支持`MONGODB-CR`，3.x版本也许可行
3. 直接在yml文件中，将连接参数替换为uri方式连接，在uri中指定

```yml
// 示例--- 用户名(该用户在admin库中,能够操作dist数据库):密码.../数据库?认证数据库&认证机制
uri: mongodb://admin:123456@127.0.0.1:27017/dist?authSource=admin&authMechanism=SCRAM-SHA-1
```

开启权限验证后，每次会话只能使用一次身份验证，即我每次在一个数据验证了身份，想要再去验证别的数据库操作别的数据库时会报错

![](https://raw.githubusercontent.com/HHHHire/HHHHire.github.io/master/_posts/images/mongo-error1.png)

需要退出，重新登录 mongo 

