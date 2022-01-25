## 参考文档([GitHub 的 MySQL 高可用性实践分享](https://www.oschina.net/translate/mysql-high-availability-at-github?print)),根据调整写出以下方案

技术细节:

- 当主库宕掉的时候,orchestrator不仅会每隔InstancePollSeconds(默认五秒)监测主库，也会检测从库(通过select round(absolute_lag) from meta.heartbeat_view"检查,配置文件里自定义语句)。比如，要诊断出主库挂了的情况，必须满足以下两个条件：orchestrator联系不到主库。登录到主库对应的从库，从库也连不上主库。
- 使用伪GTID的时候需要建一个表专门存放orchestrator生成的GTID,每次写入binlog的话会把这个表的GTID写进去,如下图

![1643075087959](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643075087959.png)

![1643075108838](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643075108838.png)

- orchestrator没有将错误按时间来进行分类，而是按复制拓扑服务器本身进行分类。当所有的从库都连不上主库的时候，说明复制拓扑被破坏，进行故障转移。
- 通过show slave hosts;命令发现实例,然后根据host:port访问实例
- 高可用有两种方式,一直是多个实例一个数据库,只有数据库没宕掉就能正常使用.一直是奇数实例,每个实例都有自己的实例,然后通过raft协议保证数据一致性

## 环境规划

| ip       |         | 端口 | 名称         |                        |
| :------- | :------ | :--- | :----------- | :--------------------- |
| 10.1.1.1 | 5.6.51  | 3306 | Master       | /var/lib/mysql/binlog/ |
| 10.1.1.2 | 5.6.51  | 3306 | Slave1       | /var/lib/mysql/binlog/ |
| 10.1.1.3 | 5.6.51  | 3306 | Slave2       | /var/lib/mysql/binlog/ |
| 10.1.1.4 | 3.2.6-1 | 3000 | orchestrator |                        |

>这里注意下,mysql版本
>
>* 5.6: I/O thread同步并发线程是以库级别并行的,也就是说两个库可以并行两个线程,三个库可以并行三个线程,但是注意一个库不要开启并行,影响性能.而且就算开启也会存在slave同步延迟
>* 5.7: I/O thread同步并发线程可以做到按行级别并行,线程数量一般与CPU数量一样,可以做到slave节点无延迟,slave_parallel_workers=4,slave_parallel_type=LOGICAL_CLOCK,这两个参数一定要加上

![1643075293181](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643075293181.png)

## 先上总结:

> 此方案实现功能
>
> - 自动故障转移
> - 支持自动修改拓扑,比如slave2同步过慢自动调整到slave1后面减少master读取负载
> - 故障转移的时候执行脚本,这点潜力无限
> - 基于go开源,可以自己二次开发
> - ui管理友好
>
> 缺点:
>
> - 学习成本高,配置复杂

| 名称                   | MySQL自己的gtid                                              | orchestrator的伪gtid                  |
| :--------------------- | :----------------------------------------------------------- | :------------------------------------ |
| 故障转移               | 支持                                                         | 支持                                  |
| 自动定位binlog断开位置 | 支持                                                         | 不支持,但是可以从数据库查到binlog位置 |
| 使用限制               | 主从库的表存储引擎必须是一致的不允许一个SQL同时更新一个事务引擎和非事务引擎的表不支持create table….select 语句复制（主库直接报错）不支持临时表 | 无限制,只需要在每个实例建个gtid表     |

## 搭建mysql

```
5.6yum源:rpm -Uvh http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
5.7yum源:rpm -Uvh http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
```

### 安装mysql

```
sudo yum clean all
sudo yum -y install mysql-community-server
```

### 添加binlog目录

```
mkdir -p /var/lib/mysql/binlog/
sudo chown mysql:mysql /var/lib/mysql/binlog/
```

### 修改配置文件

Master与Slave配置一样,只server id不同

```yaml
server-id=1 # 要唯一
binlog_row_image=FULL
binlog_format=ROW # binlog日志格式
log-bin=/var/lib/mysql/binlog/mysql-bin.log # binlog日志文件
expire_logs_days = 30 # binlog过期清理时间
max_binlog_size = 300m # binlog每个日志文件大小
binlog_cache_size = 12m # binlog缓存大小
max_binlog_cache_size = 512m # 最大binlog缓存大小
slave_sql_verify_checksum=0
log_slave_updates= 1
master_info_repository=table
relay_log_info_repository=table
report-host=10.1.1.1   # 返回给master的地址
report_port=3306       # # 返回给master的端口
slave_net_timeout=8
master_info_repository=table  # 主节点信息写表里
relay_log_info_repository=table  # slave信息写表里
relay_log_recovery=1 # 数据库启动后立即启动自动relay log恢复
slave_parallel_workers=4 # 设置I/O thread同步并发线程数量,一般与CPU数量一样
#slave_parallel_type=LOGICAL_CLOCK    # 设置并行同步粒度,库级别或者表级别,MySQL5.7支持的参数
```

启动

```
sudo systemctl start mysqld
```

初始化账户

```
mysql_secure_installation
```

### 添加账户

```
所有mysql实例添加的账户
CREATE USER 'orchestrator'@'%' IDENTIFIED BY 'qweasd';
GRANT SUPER, PROCESS, REPLICATION SLAVE, REPLICATION CLIENT, RELOAD ON *.* TO 'orchestrator'@'%';
GRANT SELECT ON meta.* TO 'orchestrator'@'%';   #如果需要伪GTID,需要导入sql
GRANT SELECT ON mysql.slave_master_info TO 'orchestrator'@'%';   # 允许读取mysql主从信息,提前设置把信息存到表里
GRANT DROP ON _pseudo_gtid_.* to 'orchestrator'@'%';   #增加更新伪gtid权限
刷新权限
FLUSH PRIVILEGES;
```

### 创建拓扑信息库

```sql
DROP TABLE IF EXISTS `cluster`;
CREATE TABLE `cluster`  (
  `anchor` tinyint(4) NOT NULL,
  `cluster_name` varchar(128) CHARACTER SET ascii COLLATE ascii_general_ci NOT NULL DEFAULT '',
  `cluster_domain` varchar(128) CHARACTER SET ascii COLLATE ascii_general_ci NOT NULL DEFAULT '',
  `data_center` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  PRIMARY KEY (`anchor`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;
INSERT INTO `cluster` VALUES (1, 'test', 'cc', 'TJ');  # 根据需求自定义
SET FOREIGN_KEY_CHECKS = 1;

DROP TABLE IF EXISTS `heartbeat_view`;
CREATE TABLE `heartbeat_view`  (
  `absolute_lag` int(11) NULL DEFAULT NULL
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_bin ROW_FORMAT = Compact;
INSERT INTO `heartbeat_view` VALUES (1);
SET FOREIGN_KEY_CHECKS = 1;
```

 如果开启伪GTID导入下面的sql 

`source pseudo-gtid.sql`

## 部署orchestrator

下载地址: [Release GA release v3.2.6 · openark/orchestrator · GitHub](https://github.com/openark/orchestrator/releases/tag/v3.2.6) 

`sudo yum localinstall -y orchestrator-3.2.6-1.x86_64.rpm`

### 再orchestrator专用mysql上创建orchestrator需要的库,表会自动生成

```sql
CREATE DATABASE IF NOT EXISTS orchestrator;
CREATE USER 'orchestrator_srv'@'%' IDENTIFIED BY 'qweasd';
GRANT ALL ON orchestrator.* TO 'orchestrator_srv'@'%';
```

### 配置orchestrator

有几个关注的配置:

```sql
ListenAddress:":3000", -- web http tpc 监听端口
BackendDB:"mysql", --后端数据库类型，可选mysql或则sqlite3
MySQLOrchestratorPort:3306, --后端数据库端口
MySQLTopologyUser: "orchestrator", --实例监控账户,所有mysql都要有这账号
MySQLTopologyPassword: "qweasd", --实例监控密码
MySQLOrchestratorHost: "10.1.1.1", --orchestrator使用的数据库地址
MySQLOrchestratorPort: 3307, --orchestrator使用的数据库端口
MySQLOrchestratorDatabase: "orchestrator", --orchestrator使用的数据库名称
MySQLOrchestratorUser: "orchestrator_srv", --orchestrator使用的账户
MySQLOrchestratorPassword: "qweasd", --orchestrator使用的密码
ReplicationLagQuery: "select round(absolute_lag) from meta.heartbeat_view", --健康检查sql
AutoPseudoGTID: true, --自动注入伪GTID
```

完整配置: 

带解释的配置:

#### 启动

```
systemctl start orchestrator
systemctl enable orchestrator
```

### 启动报错查看日志

`less /var/log/messages`
没问题的话访问http://ip:3000/
添加服务

![1643077735274](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643077735274.png)

![1643077940976](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643077940976.png)

查看

![1643077909835](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643077909835.png)

## 测试故障转移

停掉master

orchestrator上显示master断开

![1643077816765](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643077816765.png)

## orchestrator显示切换信息

![1643078029401](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643078029401.png)

可以看到切换详情

![1643078108495](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643078108495.png)

老master节点正常启动后

```
CHANGE MASTER TO MASTER_HOST='10.1.1.1',MASTER_USER='orchestrator',MASTER_PASSWORD='qweasd',MASTER_PORT=3306,MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=0;

START SLAVE;
```

查看最新拓扑图

![1643078141329](D:\zxt\Desktop\mysql 高可用\Orchestrator-MySQL\Orchestrator-MySQL.assets\1643078141329.png)