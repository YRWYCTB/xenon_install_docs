#标题： 利用xenon实现MySQL的高可用切换

目标： 现有的复制结构利用xenon提供MySQL高可用解决方案

https://github.com/radondb/xenon


0. 环境介绍
   dzst150 : 172.18.0.160:3336 master
   
   dzst151 : 172.18.0.151:3336 slave
   
   dzst152 : 172.18.0.152:3336 slave 

   服务IP： 172.18.0.200

1. MySQL安装准备

   1.1 MySQL 5.7.30 GTID 复制结构搭建
   
   1.2 MySQL 5.7.30 半同步插件加载 
   ```sql
     set global super_read_only=0;
     set global read_only=0; 
     INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
     INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
```
	 启动半同步:
	 配置文件中增加：
   
	rpl_semi_sync_slave_enabled  =1
  
	rpl_semi_sync_master_enabled =1

使用如下用户配置主从	
```sql
	CHANGE MASTER TO
	MASTER_HOST='172.18.0.152',
	MASTER_USER='tian',
	MASTER_PASSWORD='8085782',
	MASTER_PORT=3336,
	MASTER_AUTO_POSITION = 1;
```
   1.3 处理MySQL的帐号,由原来的/sbin/nologin -> /bin/bash， 修改mysql用户的密码
   
```sh
    chsh mysql
    Changing shell for mysql.
    New shell [/sbin/nologin]: /bin/bash
	Shell changed.
	
	[root@dzst160 ~]# passwd mysql
	Changing password for user mysql.
	New password: 
	BAD PASSWORD: The password is shorter than 8 characters
	Retype new password: 
	passwd: all authentication tokens updated successfully.
```
  用户名: mysql 
  
	密码：mysql 
  
	创建mysql用户的家目录
```sh
	mkdri /home/mysql
	chown -R mysql:mysql 
	```
   1.4 做基于mysql帐号的ssh信任 -> xenon ??需要看视频
   151:
```sh
   su - mysql
   ssh-keygen
   ssh-copy-id -i /home/mysql/.ssh/id_rsa.pub 172.18.0.160
   ssh-copy-id -i /home/mysql/.ssh/id_rsa.pub 172.18.0.152
   
   152:
   su - mysql
   ssh-keygen
   ssh-copy-id -i /home/mysql/.ssh/id_rsa.pub 172.18.0.160
   ssh-copy-id -i /home/mysql/.ssh/id_rsa.pub 172.18.0.151
   
   160:
   su - mysql
   ssh-keygen
   ssh-copy-id -i /home/mysql/.ssh/id_rsa.pub 172.18.0.151
   ssh-copy-id -i /home/mysql/.ssh/id_rsa.pub 172.18.0.152
```
2. xenon下载编译
   2.1 下载编译xenon
```sh
   golang -> echo "export PATH=$PATH:/usr/local/go/bin" >>/etc/profile 
   source /etc/profile 
   git clone https://github.com/radondb/xenon 
   cd xenon
   make
   mkdir xenon 
   cp -r bin xenon/
   cp -r conf/xenon-simple.conf.json xenon/xenon.json 
   echo "/etc/xenon/xenon.json" >xenon/bin/config.path 
```
   #将二进制文件和配置文件拷贝到/etc目录下
```sh
   cp -r xenon /etc/
```
	#将二进制文件和配置文件拷贝到/data目录下
```sh
	cp -r xenon /data/
	chown -R mysql:mysql /data/xenon
	```
   2.2 给运行xenon的帐号mysql添加sudo /usr/ip 权限
```sh
   visudo 
   mysql   ALL=(ALL)       NOPASSWD: /usr/sbin/ip
```
   2.3 安装xtrabackup 
   2.4 启动Xenon组成集群
    
   
  启动，需要在所有节点执行
	切换到 mysql 用户下执行
```sh
	su - mysql
	cd /data/xenon/
	
   ./bin/xenon -c /etc/xenon/xenon.json  >./xenon.log 2>&1 &
```sh
   
```json
   {
        "server":
        {
                "endpoint":"172.18.0.151:8801"
        },

        "raft":
        {
                "meta-datadir":"raft.meta",
                "heartbeat-timeout":1000,
                "election-timeout":3000,
                "leader-start-command":"sudo /sbin/ip a a 172.18.0.200/16 dev eth0 && arping -c 3 -A  172.18.0.200  -I eth0",
                "leader-stop-command":"sudo /sbin/ip a d 172.18.0.200/16 dev eth0"
        },

        "mysql":
        {
                "admin":"tian",
                "passwd":"8085782",
                "host":"127.0.0.1",
                "port":3306,
                "basedir":"/usr/local/mysql",
                "defaults-file":"/data/mysql/mysql3306/my3306.cnf",
                "ping-timeout":1000,
                "master-sysvars":"super_read_only=0;read_only=0;sync_binlog=default;innodb_flush_log_at_trx_commit=default",
                "slave-sysvars": "super_read_only=1;read_only=1;sync_binlog=1000;innodb_flush_log_at_trx_commit=2"
        },

        "replication":
        {
                "user":"tian",
                "passwd":"8085782"
        },

        "backup":
        {
                "ssh-host":"172.18.0.151",
                "ssh-user":"mysql",
                "ssh-passwd":"mysql",
                "ssh-port":22,
                "backupdir":"/data/mysql/mysql3306/data",
                "xtrabackup-bindir":"/usr/bin",
                "backup-iops-limits":100000,
                "backup-use-memory": "1GB",
                "backup-parallel": 2
        },ZZ

        "rpc":
        {
                "request-timeout":500
        },

        "log":
        {
                "level":"INFO"
        }
}
```
添加xenon的成员,需要在所有节点执行
```sh
./bin/xenoncli cluster add 172.18.0.160:8801,172.18.0.151:8801,172.18.0.152:8801

3. 检验环境
[root@dzst160 bin]# mysql -utian -p -h172.18.0.200 -P 3336
Enter password: 

Welcome to the MySQL monitor.  Commands end with ; or \g.

Your MySQL connection id is 25

Server version: 5.7.30-log MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> \s
--------------
mysql  Ver 14.14 Distrib 5.7.30, for linux-glibc2.12 (x86_64) using  EditLine wrapper

Connection id:		25
Current database:	
Current user:		tian@dzst160
SSL:			Cipher in use is ECDHE-RSA-AES128-GCM-SHA256
Current pager:		stdout
Using outfile:		''
Using delimiter:	;
Server version:		5.7.30-log MySQL Community Server (GPL)
Protocol version:	10
Connection:		172.18.0.200 via TCP/IP
Server characterset:	utf8
Db     characterset:	utf8
Client characterset:	latin1
Conn.  characterset:	latin1
TCP port:		3336
Uptime:			1 hour 21 min 30 sec

Threads: 3  Questions: 53  Slow queries: 0  Opens: 118  Flush tables: 2  Open tables: 10  Queries per second avg: 0.010
--------------

[root@dzst160 bin]# 


节点重建：
```sh
-bash-4.2$ cd /data/xenon/
-bash-4.2$ ls
bin  raft.meta  xenon.json  xenon.log
-bash-4.2$ ./bin/xenoncli mysql rebuildme
 2020/05/25 19:19:40.859070 	  [WARNING]  	=====prepare.to.rebuildme=====
			IMPORTANT: Please check that the backup run completes successfully.
			           At the end of a successful backup run innobackupex
			           prints "completed OK!".
			
 2020/05/25 19:19:40.860029 	  [WARNING]  	S1-->check.raft.leader
 2020/05/25 19:19:40.868063 	  [WARNING]  	rebuildme.found.best.slave[172.18.0.152:8801].leader[172.18.0.160:8801]
 2020/05/25 19:19:40.868160 	  [WARNING]  	S2-->prepare.rebuild.from[172.18.0.152:8801]....
 2020/05/25 19:19:40.869109 	  [WARNING]  	S3-->check.bestone[172.18.0.152:8801].is.OK....
 2020/05/25 19:19:40.869194 	  [WARNING]  	S4-->set.learner
 2020/05/25 19:19:40.870111 	  [WARNING]  	S5-->stop.monitor
 2020/05/25 19:19:40.870903 	  [WARNING]  	S6-->kill.mysql
 2020/05/25 19:19:40.903063 	  [WARNING]  	S7-->check.bestone[172.18.0.152:8801].is.OK....
 2020/05/25 19:19:40.921546 	  [WARNING]  	S8-->rm.datadir[/data/mysql/mysql3336/data]
 2020/05/25 19:19:40.921605 	  [WARNING]  	S9-->xtrabackup.begin....
 2020/05/25 19:19:40.921901 	  [WARNING]  	rebuildme.backup.req[&{From: BackupDir:/data/mysql/mysql3336/data SSHHost:172.18.0.151 SSHUser:mysql SSHPasswd:mysql SSHPort:22 IOPSLimits:100000 XtrabackupBinDir:/usr/bin}].from[172.18.0.152:8801]
 2020/05/25 19:19:45.166660 	  [WARNING]  	S9-->xtrabackup.end....
 2020/05/25 19:19:45.166689 	  [WARNING]  	S10-->apply-log.begin....
 2020/05/25 19:19:49.256462 	  [WARNING]  	S10-->apply-log.end....
 2020/05/25 19:19:49.256494 	  [WARNING]  	S11-->start.mysql.begin...
 2020/05/25 19:19:49.257334 	  [WARNING]  	S11-->start.mysql.end...
 2020/05/25 19:19:49.257351 	  [WARNING]  	S12-->wait.mysqld.running.begin....
 2020/05/25 19:19:52.268808 	  [WARNING]  	wait.mysqld.running...
 2020/05/25 19:19:52.281056 	  [WARNING]  	S12-->wait.mysqld.running.end....
 2020/05/25 19:19:52.281074 	  [WARNING]  	S13-->wait.mysql.working.begin....
 2020/05/25 19:19:55.282103 	  [WARNING]  	wait.mysql.working...
 2020/05/25 19:19:55.282531 	  [WARNING]  	S13-->wait.mysql.working.end....
 2020/05/25 19:19:55.282590 	  [WARNING]  	S14-->stop.and.reset.slave.begin....
 2020/05/25 19:19:55.294547 	  [WARNING]  	S14-->stop.and.reset.slave.end....
 2020/05/25 19:19:55.294575 	  [WARNING]  	S15-->reset.master.begin....
 2020/05/25 19:19:55.297277 	  [WARNING]  	S15-->reset.master.end....
 2020/05/25 19:19:55.297443 	  [WARNING]  	S15-->set.gtid_purged[91756561-9e39-11ea-8b54-0242ac120098:1-8,
918b0f1f-9e39-11ea-b6e6-0242ac1200a0:1-18
].begin....
 2020/05/25 19:19:55.298572 	  [WARNING]  	S15-->set.gtid_purged.end....
 2020/05/25 19:19:55.298612 	  [WARNING]  	S16-->enable.raft.begin...
 2020/05/25 19:19:55.299256 	  [WARNING]  	S16-->enable.raft.done...
 2020/05/25 19:19:55.299297 	  [WARNING]  	S17-->wait[3000 ms].change.to.master...
 2020/05/25 19:19:55.299333 	  [WARNING]  	S18-->start.slave.begin....
 2020/05/25 19:19:55.301779 	  [WARNING]  	S18-->start.slave.end....
 2020/05/25 19:19:55.301861 	  [WARNING]  	completed OK!
 2020/05/25 19:19:55.301896 	  [WARNING]  	rebuildme.all.done....
-bash-4.2$ 
```

4. 限制
  1. MySQL  5.7 版本 GTID 半同步
  2. xenon, mysql跑在同一个帐号下， 这个帐号需要有Shell
  3. 使用mysqld_safe启用mysql
