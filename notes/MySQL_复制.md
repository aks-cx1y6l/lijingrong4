# MySQL 复制

## 一、日志格式

### 1.1 二进制日志格式

MySQL 二进制日志是进行主从复制的基础，它记录了所有对 MySQL 数据库的修改事件，包括增删改查和表结构修改。当前 MySQL 一共支持三种二进制日志格式，可以通过 binlog-format 参数来进行控制，其可选值如下：

+ **STATEMENT**：段格式。是 MySQL 最早支持的二进制日志格式。其记录的是实际执行修改的 SQL 语句，因此在进行批量修改时其所需要记录的数据量比较小，但对于 UUID() 或者其他依赖上下文的执行语句，可能会在主备上产生不一样的结果。
+ **ROW**：行格式，是 MySQL 5.7 版本之后默认的二进制日志格式。其记录的是修改前后的数据。因此在批量修改时其需要记录的数据量比较大，因为需要记录每一行语句修改后的结果。但其安全性比较高，不会导致主备出现不一致的情况。同时因为 ROW 格式是在从库上直接应用更改后的数据，其还能减少锁的使用。
+ **MIXED**：是以上两种日志的混合方式，默认采用段格式进行记录，当段格式不适用时 (如 UUID() )，则默认采用 ROW 格式。

通常在主备之间网络情况良好的时，可以优先考虑使用 ROW 格式，此时数据一致性最高，其次是 MIXED 格式。在设置 ROW 格式时，还有一个非常重要的参数 binlog_row_image 。 

### 1.2 binlog_row_image

binlog_row_image 有以下三个可选值：

+ **full**：默认值，记录行在修改前后所有列的值。

+ **minimal**：只记录修改涉及列的值。

+ **noblob**：与 full 类似，但如果 BLOB 或 TEXT 列没有修改，则不对其进行记录。

binlog-format 和 binlog_row_image 的默认值可能在不同版本存在差异，可以使用以下命令进行查看。通常情况下，为了减少在主备复制中需要传输的数据量，可以将 binlog_row_image 的值设置为 minimal 或 noblob。

```sql
show variables like 'binlog_format';
show variables like 'binlog_row_image';
```

## 二、基于二进制日志的复制

### 2.1 复制原理

MySQL 的复制原理如下图所示：

+ 主库首先将变更写入到自己的二进制日志中；
+ 备库会启动一个 IO 线程，然后主动去主库节点上获取变更日志，并写入到自己的中继日志中。
+ 之后从中继日志中读取变更事件，在从库上执行变更。
+ 当备库与主库数据状态一致，备库的 IO 线程就会进入睡眠。当主库再次发生变更时，其会向备库发出信号，唤醒 IO 线程并再次进行工作。

如果没有进行任何配置，主库在将变更写入到二进制日志后，就会返回对客户端的响应，因此默认情况下的复制是完全异步进行的，主备之间可能会短暂存在数据不一致的情况。

![mysql-replication](D:\Full-Stack-Notes\pictures\mysql-replication.jpg)



### 2.2 配置步骤

首先主节点需要开启二进制日志，并且在同一个复制环境下，主备节点的 server-id  需要不一样：

```properties
[mysqld]
server-id = 226
# 开启二进制日志
log-bin=mysql-bin
```

在备份节点配置中继日志：

```properties
[mysqld]
server-id = 227
# 配置中继日志
relay_log  = mysql-relay-bin
# 为了保证数据的一致性，从节点应该设置为只读
read_only = 1
# 以下两个配置代表是否开启二进制日志，如果该从节点还作为其他备库的主节点，则开启，否则不用配置
log-bin = mysql-bin
# 是否将中继节点收到的复制事件写到自己的二进制日志中
log_slave_updates = 1
```

创建用于进行复制账号，并为其授予权限：

```shell
CREATE USER 'repl'@'192.168.0.%' IDENTIFIED WITH mysql_native_password BY '123456'; 
GRANT REPLICATION SLAVE on *.* TO 'repl'@'192.168.0.%' ;
```

查看主节点二进制日志的状态：

```shell
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      887 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

基于日志和偏移量，建立复制链路：

```shell
CHANGE MASTER TO MASTER_HOST='192.168.0.226',\
 		MASTER_USER='repl',	\
		MASTER_PASSWORD='123456',\
		MASTER_LOG_FILE='mysql-bin.000001',\
		MASTER_LOG_POS=887;
```

开始复制：

```shell
START SLAVE;
```

查看从节点复制状态，主要参数有 Slave_IO_Running 和 Slave_SQL_Running，其状态都为 Yes 表示用于进行复制的 IO 进程已经开启。Seconds_Behind_Master 参数表示从节点复制的延迟量。此时可以在主库上进行任意更改，并在备库上查看情况。

```shell
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.0.226
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 887
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 322
        Relay_Master_Log_File: mysql-bin.000001
        #    Slave_IO_Running: Yes
        #   Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 887
              Relay_Log_Space: 530
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
     #  Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 226
   #              Master_UUID: e1148574-bdd0-11e9-8873-0800273acbfd
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:
1 row in set (0.00 sec)
```

### 2.3 优缺点

基于二进制日志的复制是 MySQL 最早使用的复制技术，因此 MySQL 对其的支持比较完善，对执行修改的 SQL 语句几乎没有任何限制。其主要的缺点是在一主多从的高可用复制架构中，如果主库发生宕机，此时想要自动通过从库的日志和偏移量来确定新的主库比较困难。

## 三、基于 GTID 的复制

### 2.1 GTID 

MySQL 5.6 版本之后提供了一个新的复制模式：基于 GTID 的复制。GTID 全称为 Global Transaction ID，即全局事务 ID。它由每个服务节点的唯一标识和其上的事务 ID 共同组成，格式为： *server_uuid : transaction_id* 。通过 GTID ，可以保证在主库上的每一个事务都能在备库上得到执行，不会存在任何疏漏。 

### 2.2 配置步骤

主从服务器均增加以下 GTID 的配置：

```properties
gtid-mode = ON
# 防止执行不受支持的语句，下文会有说明
enforce-gtid-consistency = ON
```

如果配置过上面的基于二进制日志的复制，还需要在从服务器上执行以下命令，关闭原有复制链路：

```shell
STOP SLAVE IO_THREAD FOR CHANNEL '';
```

建立新的基于 GTID 复制链路，指定 MASTER_AUTO_POSITION = 1 表示由程序来自动确认开始同步的 GTID 的位置：

```shell
CHANGE MASTER TO MASTER_HOST='192.168.0.226',\
		MASTER_USER='repl',
		MASTER_PASSWORD='123456',
		MASTER_AUTO_POSITION=1;
```

开始复制：

```shell
START SLAVE;
```

在主节点上执行任意修改操作，并查看从节点状态，关键的输出如下：Retrieved_Gtid_Set 代表从主节点上接收到的两个事务，Executed_Gtid_Set 表示这两个事务已经在从库上得到执行。

```sql
mysql> SHOW SLAVE STATUS\G
....
Master_UUID			: e1148574-bdd0-11e9-8873-0800273acbfd
Retrieved_Gtid_Set	: e1148574-bdd0-11e9-8873-0800273acbfd:1-2
Executed_Gtid_Set	: e1148574-bdd0-11e9-8873-0800273acbfd:1-2
.....
```

### 2.3 优缺点

GTID 复制的优点在于程序可以自动确认开始复制的 GTID 点。但其仍然存在以下限制：

+ 不支持 CREATE TABLE ... SELECT 语句。  因为在 ROW 格式下，该语句将会被记录为具有不同 GTID 的两个事务，此时从服务器将无法正确处理。

+ 事务，过程，函数和触发器内部的 CREATE TEMPORARY TABLE 和 DROP TEMPORARY TABLE 语句均不受支持。

为防止执行不受支持的语句，建议配置和上文配置一样，开启 enforce-gtid-consistency 属性， 开启后在主库上执行以上不受支持的语句都将抛出异常并提示。

## 四、半同步复制



## 五、高可用架构

不论是主主复制结构，还是一主多从复制结构，单存依靠复制只能解决数据可靠性的问题，并不能解决系统高可用的问题，想要保证高可用，系统必须能够自动进行故障转移，即在主库宕机时，主动将其它备库升级为主库。常用的有以下两种解决方案：

### 4.1 MMM

1. Monitor 检测到 Master1 连接失败；
2. Monitor 发送 set_offline 指令到 Master1 的 Agent；
3. Master1 Agent 如果存活，下线写 VIP，尝试把 Master1 设置为 read_only=1；
4. Moniotr 发送 set_online 指令到 Master2；
5. Master2 Agent 接收到指令，执行 `select master_pos_wait()` 等待同步完毕；
6. Master2 Agent 上线写 VIP，把 Master2 节点设为 read_only=0；
7. Monitor 发送更改同步对象的指令到各个 Slave 节点的 Agent；
8. 各个 Slave 节点向新 Master 同步数据；

简单来说，MMM 就是暴力的将 Master1 切换到 Master2 ，它本生不会尝试去补齐丢失的数据。

### 4.2 MHA

1. 尝试从宕机 Master 中保存二进制日志；
2. 找到含有最新中继日志的 Slave；
3. 把最新中继日志应用到其他实例，实现各实例数据一致；
4. 应用从 Master 保存的二进制日志事件；
5. 提升一个 Slave 为 Master ；
6. 其他 Slave 向该新 Master 同步。

当 Master 宕机后， MHA 会尝试保存宕机 Master 的二进制日志，然后自动判断 MySQL 集群中哪个实例的中继日志是最新的，并将有最新日志的实例的差异日志传到其他实例补齐，从而实现所有实例数据一致。然后把宕机 Master 的二进制日志应用到选定节点，并提升为 Master 。

从切换流程流程可以看到，如果宕机 Master 主机无法 SSH 登录，那么第一步就没办法实现，对于 MySQL5.5 以前的版本，数据还是有丢失的风险。对于 5.5 后的版本，开启**半同步复制**后，真正有助于避免数据丢失，半同步复制保证至少一个 （不是所有）slave 在 master 提交时接收到二进制日志事件。因此，对于可以处理一致性问题的 MHA 可以实现"几乎没有数据丢失"和"从属一致性"。

### 优点

- 开源，用 Perl 编写
- 方案成熟，故障切换时，MHA 会做日志补齐操作，尽可能减少数据丢失，保证数据一
- 部署不需要改变现有架构

### 限制

- 各个节点要打通 SSH 信任，有一定的安全隐患
- 没有 Slave 的高可用
- 自带的脚本不足，例如虚IP配置需要自己写命令或者依赖其他软件
- 需要手动清理中继日志





## 参考资料

+ [MHA高可用架构](https://segmentfault.com/a/1190000017486693)

+ [MMM高可用架构](https://segmentfault.com/a/1190000017286307)

+ 