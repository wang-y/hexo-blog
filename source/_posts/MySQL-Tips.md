---
title: MySQL Tips
date: 2018-01-19 11:18:51
tags:
    - MySQL
    - 主从同步
    - command
---

# 存储过程示例

<!-- more -->

````sql
##创建存储过程##
DROP PROCEDURE IF EXISTS test;
DELIMITER //
CREATE PROCEDURE test(OUT $cards LONGTEXT)
	#子句中存在修改语句
	MODIFIES SQL DATA
	BEGIN
		#定义变量

		#定义卡号
		DECLARE $cardCode VARCHAR(32);
		#定义申请日期
		DECLARE $applyDate DATETIME;
		#定义发卡日期
		DECLARE $handoutDate DATETIME;
		#定义激活日期
		DECLARE $activeDate DATETIME;
		#定义到期日期
		DECLARE $dueDate DATETIME;
		#定义服务到期日期
		DECLARE $endDate DATETIME;
		#定义停用日期
		DECLARE $stopDate DATETIME;
		#定义当前日期
		DECLARE $nowDate DATETIME;
		#定义卡状态值
		DECLARE $status INT;
		
		#定义返回修改结果
		DECLARE $cardCodes LONGTEXT;
		
		#这个用于处理游标到达最后一行的情况
		DECLARE flag INT DEFAULT 0;
		
		#标记是否出错
		DECLARE t_error INT DEFAULT 0;

		#声明游标cursor_name(cursor_name是个多行结果集) 
		DECLARE cursor_name CURSOR FOR SELECT cardCode,applyDate,handoutDate,activeDate,dueDate,stopDate,feeFlag FROM core_bd_card_info WHERE feeFlag IN(2,3);
		
		#设置一个终止标记
		DECLARE CONTINUE HANDLER FOR SQLSTATE '02000' SET flag =1;
		
		#如果出现sql异常，则将t_error设置为1后继续执行后面的操作 
		DECLARE CONTINUE HANDLER FOR SQLEXCEPTION SET t_error=1; 
		
		#打开游标 
		OPEN cursor_name;
		
		#获取游标当前指针的记录，读取一行数据并传给变量
		FETCH cursor_name INTO $cardCode,$applyDate,$handoutDate,$activeDate,$dueDate,$stopDate,$status;
		
		SET $cardCodes='@@@';

		#开启事务	
		START TRANSACTION;
		 
		#开始循环，判断是否游标已经到达了最后作为循环条件
		WHILE flag <> 1 DO
			#格式化日期
			SET $endDate = DATE_FORMAT(DATE_ADD($dueDate, INTERVAL 30 DAY),'%Y-%m-%d');
			SET $dueDate = DATE_FORMAT($dueDate,'%Y-%m-%d');
			SET $nowDate = DATE_FORMAT(curdate(),'%Y-%m-%d');
			#卡状态为2
			IF $status=2
			THEN
				#如果当前日期大于到期日期且小于服务停止日期
				IF $nowDate>$dueDate AND $nowDate<=$endDate
				THEN
					#更新feeFlag为3
					UPDATE core_bd_card_info SET feeFlag=3 WHERE cardCode=$cardCode;
					SET $cardCodes=CONCAT($cardCodes,',_',$cardCode);
				#如果当前日期大于服务停止日期
				ELSEIF $nowDate>$endDate
				THEN
					#更新feeFlag为4
					UPDATE core_bd_card_info SET feeFlag=4 WHERE cardCode=$cardCode;
					UPDATE core_qryaccount_card_relation SET feeFlag = 1 WHERE cardCode=$cardCode;
					UPDATE core_pushclient_card_relation SET feeFlag = 1 WHERE cardCode=$cardCode;
					SET $cardCodes=CONCAT($cardCodes,',$',$cardCode);
				END IF;
			#卡状态为2
			ELSEIF $status=3
			THEN
				#如果当前日期大于服务停止日期
				IF $nowDate>$endDate
				THEN
					#更新feeFlag为4
					UPDATE core_bd_card_info SET feeFlag=4 WHERE cardCode=$cardCode;
					UPDATE core_qryaccount_card_relation SET feeFlag = 1 WHERE cardCode=$cardCode;
					UPDATE core_pushclient_card_relation SET feeFlag = 1 WHERE cardCode=$cardCode;
					SET $cardCodes=CONCAT($cardCodes,',$',$cardCode);
				END IF;
			END IF; #结束判断
			#读取下一行游标数据
			FETCH  cursor_name INTO $cardCode,$applyDate,$handoutDate,$activeDate,$dueDate,$stopDate,$status;
		END WHILE; #结束循环
		
		#关闭游标
		CLOSE cursor_name;
		
		#标记被改变,表示事务应该回滚
		IF t_error=1
		THEN
			#事务回滚
			ROLLBACK;
			SELECT 'ERROR';
			SELECT 'ERROR' INTO $cards;
		ELSE
			#事务提交
			COMMIT;
			#_cardCode -->即将到期；$cardCode -->已经到期
			SELECT $cardCodes;
			SELECT $cardCodes INTO $cards;
		END IF;
		
	END //
DELIMITER ;
````

# MySQL主从同步 && 读写分离

## 1.主(master)从(slave)服务器上安装mysql;
## 2.配置步骤
### 主服务器master 配置：

&#160;&#160;&#160;&#160;mysql 配置文件添加：(windows: my.ini;linux:my.cnf)
***

	server-id=129	#唯一标示位，通常是设置服务器IP的末尾号
	log-bin=master-bin	#SLAVE会基于此LOG-BIN来做REPLICATION
	log-bin-index=master-bin.index #指定索引文件，此文件指示当前使用了哪个日志文件
	binlog-do-db=springjpa	#只记录指定库的更新到BINLOG，多库以逗号分隔，或者追加binlog-do-db；

&#160;&#160;&#160;&#160;在Master MySQL上创建一个用户‘repl’，并允许其他Slave服务器可以通过远程访问Master，通过该用户读取二进制日志，实现数据同步。
***

	create user repl; #创建用户
	GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.0.%' IDENTIFIED BY 'mysql'; #开放SLAVE权限给REPL，且允许的192.168.0.1~255 以此用户连接，密码设置为MYSQL。
	
&#160;&#160;&#160;&#160;配置完成后重启mysql服务

&#160;&#160;&#160;&#160;运行 > show master status\G;
***
&#160;&#160;&#160;&#160;查看master运行状态；记录File名称，及pos值；
	
&#160;&#160;&#160;&#160;将master服务器 中指定的数据库数据备份。

### 从服务器slave 配置
	
&#160;&#160;&#160;&#160;将master服务器中备份的数据库还原至从服务器数据库。
	
&#160;&#160;&#160;&#160;mysql 配置文件中添加 (windows: my.ini;linux:my.cnf)
***

	server_id   = 75	#唯一标示位，通常是设置服务器IP的末尾号
	replicate-do-db=springjpa	#只替换指定数据库，多库以逗号分隔，或追加replicate-do-db，顺序**必须**与MASTER中的binlog-do-db严格对应；
	relay-log=slave-relay-bin	#服务器I/O线程将主服务器的二进制日志读取过来记录到从服务器本地文件
	relay-log-index=slave-relay-bin.index	#本地文件的索引，此文件指示当前使用了哪个日志文件

&#160;&#160;&#160;&#160;重启mysql服务
	
&#160;&#160;&#160;&#160;重启后执行以下语句：
***
	CHANGE MASTER TO
	MASTER_HOST='172.16.232.129',	#MASTER服务器IP
    MASTER_PORT=3306,    #MASTER服务器端口
	MASTER_USER='reply_user',	#MASTER服务器开放的用户
	MASTER_PASSWORD='root',	#MASTER服务器开放的用户密码
	MASTER_LOG_FILE='master-bin.000003',  #MASTER服务器产生的日志
	MASTER_LOG_POS=154;  #POS位置

	
&#160;&#160;&#160;&#160;运行成功后执行
	> start slave；	#启动SLAVE
    > show slave status\G;#查看状态
    > stop slave;	#停止SLAVE
## 3.读写分离方案优化

* 你需要事务支持吗？ 
* 你需要全文索引吗？
* 你经常使用什么样的查询模式？ 

&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;思考上面这些问题可以让你找到合适的方向，但那并不是绝对的。如果你需要事务处理，那么InnoDB 可能是比较好的方式。如果你需要全文索引，那么通常来说 MyISAM是好的选择，因为这是系统内建的。

&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;事务支持，是一个影响你选择什么样存储引擎的重要因素，事务支持趋向于选择InnoDB方式，因为其支持事务处理和故障恢复。另外InnoDB可以利用事务日志进行数据恢复，这会比MyISAM快很多。而MyISAM可能会需要几个小时甚至几天来干这些事，InnoDB 只需要几分钟。

&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;当然操作数据库表的习惯可能也会是一个对性能影响很大的因素。比如： COUNT() 在 MyISAM 表中会非常快，而在InnoDB 表下可能会很痛苦。另外MyISAM存储引擎在筛选大量数据时非常迅速，这是它最突出的优点。MyISAM还提供了大量的特性，包括全文索引、压缩、空间函数(GIS)等，但MyISAM不支持事务和行级锁，有一个毫无疑问的缺陷就是崩溃后无法安全恢复。在读多写少的业务，如果不介意MyISAM的崩溃恢复问题，选用MyISAM引擎是合适的。

* MyISAM管理非事务表。它提供高速存储和检索，以及全文搜索能力。如果应用中需要执行大量的SELECT查询，那么MyISAM是更好的选择。
* InnoDB用于事务处理应用程序，具有众多特性，包括ACID事务支持。如果应用中需要执行大量的INSERT或UPDATE操作，则应该使用InnoDB，这样可以提高多用户并发操作的性能。

> > > 读写分离必然会分库，一个负责写入数据的数据库，一个负责读取数据的数据库，此时我们可以将负责写入的数据库采用InnoDB作为表引擎，以便控制事务；在负责读取的数据使用MyISAM作为表引擎，以提升检索和读取数据速度。

1. 在主库中创建表A，引擎选择Innodb；
2. 删除从库中自动创建的表A；
3. 在从库中创建表A，引擎选择MyISAM；


***

## 附.已有数据库配置主从同步的准备工作
 1. 在在MASTER中执行 > FLUSH TABLES WITH READ LOCK;  #锁定数据库表，权限设置为只读；
 2. 执行cmd命令或者shell命令 >mysqldump -uuser -ppassword --databases db1 [db2 db3 ...] -->parth/mysql.sql;  #备份指定数据库；
 3. 在从库中创建数据库，进入数据库后，执行导入MASTER备份 > source path/mysql.sql;  #在从库中导入主库数据；
 4. 配置主从同步，启动slave；
 5. 在MASTER中执行 > UNLOCK TABLES;  #解除数据库表锁定，恢复读写权限  

# MySQL常用命令行

## 删除表

```sql
mysql -uroot -pxxxxxxxxxxxx -N -s information_schema -e "SELECT CONCAT('DROP TABLE ',TABLE_NAME,';') FROM TABLES WHERE TABLE_SCHEMA='test_schema'" | mysql -uroot -pxxxxxxxxxxxx -f test_schema
```

## 执行sql脚本

```sql
mysql -uroot -pxxxxxxxxxxxx -Dtest_schema < /home/test_schema.sql
```

## 导出数据库

### 仅结构

```sql
mysqldump -h localhost -uroot -p123456  -d test_schema > dump.sql  # 整个数据库结构

mysqldump -h localhost -uroot -p123456  -d test_schema test_table > dump.sql  # 指定表结构
```

### 仅数据

```sql
mysqldump -h localhost -uroot -p123456  -t test_schema  > dump.sql  # 整个数据库数据

mysqldump -h localhost -uroot -p123456  -t test_schema test_table> dump.sql  # 指定表数据

```

### 导出结构+数据

```sql
mysqldump -h localhost -uroot -p123456 test_schema > dump.sql  #整个数据库结构和数据

mysqldump -h localhost -uroot -p123456 test_schema test_table > dump.sql  #指定表数据结构和数据
```
