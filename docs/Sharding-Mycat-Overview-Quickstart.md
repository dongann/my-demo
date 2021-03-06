分库分表框架系列：
- [Mycat分库分表概览](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/Sharding-Mycat-Overview-Quickstart.md)
- [Sharding-Proxy分库分表概览](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/Sharding-Sharding-Proxy-Overview-Quickstart.md)
- [DRDS产品概览](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/Sharding-DRDS-Overview.md)

-----------------------------------
#### 项目情况
MyCat基于阿里早期开源的分库分表组件Cobar改进而来，以下参考[Cobar的架构与实践](https://blog.csdn.net/jiao_fuyou/article/details/15809999)：
- [Amoeba](https://sourceforge.net/projects/amoeba)：阿里B2B开发的分库分表中间件，应该开始于2006年左右，2008年开源，2010年已经广泛运用于阿里B2B业务，目前项目停滞；<br />
  <img src="https://richie-leo.github.io/ydres/img/10/120/1016/amoeba-architecture.jpg" style="max-width:430px;" />
- [Cobar](https://github.com/alibaba)：Amoeba进化版，后端由JDBC Driver改为原生MySQL通信协议，2011年10月发布，2012年开源在code.alibabatech上，后来迁移到github，目前项目停滞；<br />
  <img src="https://richie-leo.github.io/ydres/img/10/120/1016/cobar-architecture.jpg" style="max-width:530px;" />
- Mycat：由leaderus等人基于Cobar改进发展而来，后端由BIO改为NIO，改掉一些重要bug，功能增强（增加对Order By、Group By、limit等聚合功能的支持）。当年社区比较活跃，目前活跃度一般；
- 2012年阿里云将TDDL和Cobar结合开发DRDS；

-----------------------------------
#### Mycat架构
<img src="https://richie-leo.github.io/ydres/img/10/120/1016/mycat-architecture.jpg" style="max-width:460px;" />

- Mycat实现了MySQL协议，MySQL命令行客户端、任何开发语言都能像直接连MySQL一样连接Mycat，对客户端透明，支持所有开发语言；
- Mycat解析SQL语句，根据SQL参数和分片规则进行路由，跨分片查询对结果集进行汇总、重排序、分页、聚合等，将分库分表、读写分离等数据存储伸缩方案与应用隔离；

-----------------------------------
#### Mycat分布式事务
只支持MySQL XA分布式事务，但MySQL 5.7之前的版本XA事务有问题（例如PREPARE命令没有写入binlog，发生故障主从切换时造成数据不一致），只能在5.7之后的版本上使用。

Mycat基于MySQL XA实现了TC、TM功能，主要处理方式如下：
1. 客户端执行`set xa = on`命令开启XA事务：<br />
   Mycat收到`set xa = on`命令后，在当前session中生成一个全局事务ID `xaTxId`，不做其它处理，参考[SetHandler](https://github.com/MyCATApache/Mycat-Server/blob/1.6/src/main/java/io/mycat/server/handler/SetHandler.java#L83)。<br />
   这表示当前会话在Mycat-Server上开启了XA事务，但此时Mycat还不会在后端MySQL上开启XA事务。
2. 客户端执行DML SQL语句：<br />
   Mycat-Server开启XA后，在每个`dataNode`上执行第1个更新语句（如果`strictTxIsolation`设为true，则是执行第一个查询语句）时，向MySQL发送`XA START xaTxID`语句，在MySQL节点上开启XA事务，事务状态为`TX_STARTED_STATE`，参考[MySQLConnection.synAndDoExecute(...)](https://github.com/MyCATApache/Mycat-Server/blob/1.6/src/main/java/io/mycat/backend/mysql/nio/MySQLConnection.java#L428)；
   > Mycat有一个特性，尽量延迟在MySQL中开启XA事务。客户端开启XA后，执行查询操作不会触发Mycat向MySQL开启XA，只有在第一次执行更新操作时才开启。这样做的目的是尽量重用后端连接，但无法满足`REPEADABLE READ`，可以通过[server.xml](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/server.xml#L33)中的`strictTxIsolation`关闭这个特性。<br />
   > `strictTxIsolation`为`false`时：<br />
     <img src="https://richie-leo.github.io/ydres/img/10/120/1016/strict-isolation-false.jpg" style="max-width:560px;" /> <br />
   > `strictTxIsolation`为`true`时（这种情况Mycat按下面图示占用后端连接，并且禁用读写分离强制读主库等，客户端可以用`select ... for update`加锁，实现`REPEADABLE READ`隔离级别）：<br />
     <img src="https://richie-leo.github.io/ydres/img/10/120/1016/strict-isolation-true.jpg" style="max-width:560px;" />
3. 客户端执行事务提交：<br />
   - 如果只操作了单个MySQL节点，则无需TC协调，按顺序执行`XA END xaTxId`、`XA PREPARE xaTxId`、`XA COMMIT xaTxId`，参考[NonBlockingSession.commit()](https://github.com/MyCATApache/Mycat-Server/blob/1.6/src/main/java/io/mycat/server/NonBlockingSession.java#L225)、[CommitNodeHandler](https://github.com/MyCATApache/Mycat-Server/blob/1.6/src/main/java/io/mycat/backend/mysql/nio/handler/CommitNodeHandler.java)。
   - 如果操作了多个MySQL节点，则在[MultiNodeCoordinator](https://github.com/MyCATApache/Mycat-Server/blob/1.6/src/main/java/io/mycat/backend/mysql/nio/handler/MultiNodeCoordinator.java)中处理（入口方法`executeBatchNodeCmd`）。<br />
     主要协调过程为：
     1. 向所有MySQL节点发送`XA END xaTxId`、`XA PREPARE xaTxId`命令。
     2. PREPARE全部成功，则向所有节点发送`XA COMMIT xaTxId`命令；如果有节点PREPARE失败，则回滚。
     3. 所有节点COMMIT成功，给客户端返回提交成功；如果有节点COMMIT失败，则重试，包括收到失败消息后立即重试、Mycat启动时根据协调日志的事务状态进行重试等。

Mycat实现TC功能时，将XA事务状态保存在内存和本地文件中，其结构外层为XA事务ID，内层为参与者RM信息和事务状态，用于XA过程发生异常时，可以根据TC日志回滚或重试。

Mycat事务管理方案和代码比较简单，不够严谨，例如：
- 方案方面：TC日志存本地文件，Mycat多实例组建集群会有问题；
- 代码方面：[MultiNodeCoordinator.okResponse(...) #L259](https://github.com/MyCATApache/Mycat-Server/blob/1.6/src/main/java/io/mycat/backend/mysql/nio/handler/MultiNodeCoordinator.java#L259)，收到节点`PREPARE`成功消息时，只更新内存TC日志，没有写文件，如果异常重启会造成TC日志丢失（这行代码注释掉，估计是写文件没有并发控制会造成问题）；

-----------------------------------
#### 演示方案说明
##### 表结构及拆分方案
使用[my-demo](https://github.com/liuzhibin-cn/my-demo)项目作为演示，运行项目查看演示效果。

表结构：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1016/mycat-table-schema.png" style="max-width:560px;" />

拆分情况：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1016/mycat-sharding-tables.png" style="max-width:400px;" />

- 逻辑库`db_user`：
  - `usr_user`: 会员表
    - _分片字段_：`user_id`
    - _分片规则_：`(user_id % 32) => { 0-15: dn1, 16-31: dn2 }`
    - _数据节点_：`dn1`, `dn2`
  - `usr_user_account`: 会员登录账号表 
    - _分片字段_：`account_hash`（`account.hashCode()`的绝对值）
    - _分片规则_：`(account_hash % 2) => { 0: dn1, 1: dn2 }`
    - _数据节点_：`dn1`, `dn2` 
- 逻辑库`db_order`：
  - `ord_order`: 订单主表。`order_id`创建订单时由订单服务在应用代码中生成；
    - _分片字段_：`order_id`
    - _分片规则_：`(order_id % 32) => { 0-7: dn1, 8-15: dn2, 16-23: dn3, 24-31: dn4 }`
    - _数据节点_：`dn1`, `dn2`, `dn3`, `dn4`
  - `ord_order_item`：订单明细表，与`ord_order`以父子关系表进行分片（Mycat ER分片），分片情况同`ord_order`
  - `ord_user_order`: `user_id`与`order_id`的自定义索引表，用于查询会员订单列表
    - _分片字段_：`user_id` 
    - _分片规则_：`(user_id % 32) => { 0-7: dn1, 8-15: dn2, 16-23: dn3, 24-31: dn4 }` 
    - _数据节点_：`dn1`, `dn2`, `dn3`, `dn4`

##### 物理部署
`dn0~4`可以分别部署在不同机器的MySQL实例上，演示项目方便起见只使用了一个MySQL实例，多实例部署只需修改配置文件中`dataNode`与`dataHost`映射关系即可。

-----------------------------------
#### 部署Mycat Server
[Mycat](http://www.mycat.io/)版本[1.6.7.3](http://dl.mycat.io/1.6.7.3/)，Mycat配置文件参考[docs/mycat-conf](https://github.com/liuzhibin-cn/my-demo/tree/master/docs/mycat-conf)

##### 配置
使用了Mycat数据库方式的全局序列，SQL脚本参考[sql-schema.sql](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/sql-schema.sql)。

- [server.xml](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/server.xml)：配置服务器参数，配置逻辑用户名密码：
  ```xml
  <!-- 为Mycat逻辑库定义用户、密码 -->
  <user name="mydemo" defaultAccount="true">
    <property name="password">mydemo</property>
    <property name="schemas">db_user,db_order</property>
  </user>
  ```
- [schema.xml](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/schema.xml)：配置dataHost、dataNode、逻辑schema：
  ```xml
  <mycat:schema xmlns:mycat="http://io.mycat/">
    <schema name="db_order" checkSQLschema="false" sqlMaxLimit="100">
	    <table name="ord_order" primaryKey="order_id" dataNode="dn$1-4" rule="order-rule">
		    <!-- 主键order_item_id使用Mycat全局序列，设置autoIncrement后可以像MySQL自增字段一样，
			     insert时不指定这个列，且支持last_insert_id()函数，避免在SQL中使用next value for MYCATSEQ_XXX，导致druid报错 -->
		    <childTable name="ord_order_item" primaryKey="order_item_id" autoIncrement="true" joinKey="order_id" parentKey="order_id" />
	    </table>
	    <table name="ord_user_order" primaryKey="id" autoIncrement="true" dataNode="dn$1-4" rule="user-order-rule" />
	    <!-- Seata的回滚表，Seata要求回滚表在业务库中，因此必须添加到mycat逻辑库中 -->
	    <table name="undo_log" dataNode="dn1" primaryKey="id" />
    </schema>
    <schema name="db_user" checkSQLschema="false" sqlMaxLimit="100">
		<table name="usr_user" primaryKey="user_id" autoIncrement="true" dataNode="dn1, dn2" rule="user-rule" />
		<table name="usr_user_account" primaryKey="account" dataNode="dn1, dn2" rule="user-account-rule" />
		<!-- Seata的回滚表，Seata要求回滚表在业务库中，因此必须添加到mycat逻辑库中 -->
		<table name="undo_log" dataNode="dn1" primaryKey="id" />
    </schema>
    <dataNode name="dn0" dataHost="db1" database="mydemo-dn0" />
    <dataNode name="dn1" dataHost="db1" database="mydemo-dn1" />
    <dataNode name="dn2" dataHost="db1" database="mydemo-dn2" />
    <dataNode name="dn3" dataHost="db2" database="mydemo-dn3" />
    <dataNode name="dn4" dataHost="db2" database="mydemo-dn4" />
    <dataHost name="db1" maxCon="5" minCon="5" balance="0" writeType="0" dbType="mysql" dbDriver="native">
		<heartbeat>select user()</heartbeat>
		<writeHost host="localhost" url="localhost:3306" user="root" password="1234" />
    </dataHost>
    <dataHost name="db2" maxCon="5" minCon="5" balance="0" writeType="0" dbType="mysql" dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="127.0.0.1" url="127.0.0.1:3306" user="root" password="1234" />
    </dataHost>
  </mycat:schema>
  ```
- [rule.xml](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/rule.xml)：配置分片规则：
  ```xml
  <mycat:rule xmlns:mycat="http://io.mycat/">
	  <tableRule name="order-rule">
		<rule>
			<columns>order_id</columns>
			<algorithm>order-func</algorithm>
		</rule>
	  </tableRule>
	  <tableRule name="user-order-rule">
		<rule>
			<columns>user_id</columns>
			<algorithm>user-order-func</algorithm>
		</rule>
	  </tableRule>
	  <tableRule name="user-rule">
		<rule>
			<columns>user_id</columns>
			<algorithm>user-func</algorithm>
		</rule>
	  </tableRule>
	  <tableRule name="user-account-rule">
		<rule>
			<columns>account_hash</columns>
			<algorithm>user-account-func</algorithm>
		</rule>
	  </tableRule>
	  <function name="order-func" class="io.mycat.route.function.PartitionByPattern">
		  <property name="patternValue">32</property> <!-- 对32求模 -->
		  <!-- 求模结果按照文件配置的规则映射到分片 -->
		  <property name="mapFile">order-partition.txt</property> 
	  </function>
	  <function name="user-order-func" class="io.mycat.route.function.PartitionByPattern">
		  <property name="patternValue">32</property>
		  <property name="mapFile">order-partition.txt</property> <!-- 与订单公用分片规则 -->
	  </function>
	  <function name="user-func" class="io.mycat.route.function.PartitionByPattern">
		  <property name="patternValue">32</property> <!-- 对32求模 -->
		  <property name="mapFile">user-partition.txt</property> <!-- 求模结果按照文件配置的规则映射到分片 -->
	  </function>
	  <function name="user-account-func" class="io.mycat.route.function.PartitionByMod">
		  <property name="count">2</property> <!-- 数据分为2片 -->
	  </function>
  </mycat:rule>
  ```
- [order-partition.txt](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/order-partition.txt)，配置分片映射关系：
  ```
  0-7=0
  8-15=1
  16-23=2
  24-31=3
  ```
- [sequence_db_conf.properties](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/sequence_db_conf.properties)，配置Mycat全局序列位于哪个dataNode：
  ```
  GLOBAL=dn0
  ORD_ORDER_ITEM=dn0
  ORD_USER_ORDER=dn0
  USR_USER=dn0
  ```

##### Windows环境部署
Windows环境下载[Mycat-server-1.6.7.3-release-20190927161129-win.tar.gz](http://dl.mycat.io/1.6.7.3/20190927161129/Mycat-server-1.6.7.3-release-20190927161129-win.tar.gz)

有2种运行方式：
- 通过`bin\startup_nowrap.bat`在命令行直接运行。
  > 注意：需要在命令行进入`bin`目录后再执行`startup_nowrap.bat`命令，否则会将命令行所处当前目录作为Mycat主目录，导致无法找到`lib`等目录，`classpath`无法加载jar文件。
- 通过`bin\mycat.bat install`注册为Windows服务（以管理员身份运行），开机自动启动。
  > 注意：
  > 1. 需要修改`conf\wrapper.conf`文件，将`wrapper.java.command=java`改为全路径，例如`wrapper.java.command=E:\dev-tools\java\bin\java`，否则服务启动时报无法找到`java`命令，启动失败；
  > 2. 在`conf\wrapper.conf`中配置Mycat JVM启动参数；

##### Mac/Linux环境部署
```sh
bin/mycat start      # 启动
bin/mycat stop       # 停止
bin/mycat console    # 前台运行
bin/mycat status     # 查看启动状态
```

-----------------------------------
#### Mycat管理
Mycat启动之后，`8066`为数据端口；`9066`为管理端口，不能操作数据，只能执行Mycat管理命令。连接Mycat使用`server.xml`文件中定义的用户名和密码。
> Mac环境连接Mycat必须指定TCP协议，否则会直接连接mysql的3306端口而不是Mycat，没有任何错误信息：
> ```sh
> mysql -h localhost -P 8066 -uroot -p --protocol=TCP
> mysql -h localhost -P 9066 -uroot -p --protocol=TCP
> ```

管理端口登录，通过`show @@help;`查看Mycat提供的管理命令。

-----------------------------------
#### Mycat使用
在[my-demo](https://github.com/liuzhibin-cn/my-demo)项目中使用`Mycat`，只需要改为Mycat的JDBC连接参数，包括端口号、账号密码、数据库名称等，相关属性都定义在`parent pom`中了。

`my-demo`项目配置了`maven profile`来启用Mycat，为`package.sh`指定`-mycat`选项打包即可：`package.sh -mycat`

-----------------------------------
#### 备注
- Mycat 2.0在开发中，参考[Mycat2](https://github.com/MyCATApache/Mycat2) <br />
  从新特性来看，结果集缓存、自动集群管理、支持负载均衡等主要特性没有必要由Mycat管理，使用第三方即可。
- 简单性能对比测试 <br />
  Mac book pro，单机测试，50并发线程，对相同的业务逻辑功能（使用手机号+密码注册会员）进行测试，TPS指被测试业务逻辑的每秒执行次数（包含`select from usr_user_account` + `insert into usr_user` + `insert into usr_user_account`）：
  - Mycat + MyBatis，分片: TPS在2200上下波动；
  - MyBatis，不分片: TPS在2600上下波动；
  - 纯JDBC，不分片: TPS在3400上下波动；<br />
  单机测试，Mycat server的CPU占用对测试结果有一定影响。<br />
  从结果看中间加一层mycat后性能有一定下降，但幅度不大，不及MyBatis与原生JDBC之间的差异。