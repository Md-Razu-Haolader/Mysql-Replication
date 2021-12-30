# Mysql-Replication

https://www.technhit.in/setup-mysql-replication-master-slave-mode/
Master 1 IP Address : 192.168.11.222
Master 2 IP Address : 192.168.11.221

Database to replicate : testRnd

STEP:1
===========
Create a user with replication privileges on both server:

mysql -u root -p
Enter Password:

On master 1 (192.168.11.222): 
----------------------------

create user 'rep'@'192.168.11.222' identified by 'rep123';
GRANT REPLICATION SLAVE ON *.* TO 'rep'@'192.168.11.222' IDENTIFIED BY 'rep123';
FLUSH PRIVILEGES;

On master 2 (192.168.11.221): 
----------------------------
CREATE USER 'rep'@'192.168.11.221' IDENTIFIED BY 'rep123';
GRANT REPLICATION SLAVE ON *.* TO 'rep'@'192.168.11.221' IDENTIFIED BY 'rep123';
FLUSH PRIVILEGES;

STEP:2
============
Configure MySQL nodes

On master 1 (192.168.11.222):
-----------------------------

On master 1 make changes in /usr/local/mysql/etc/my.cnf configuration:

server-id = 1

replicate-do-db = testRnd

binlog-do-db=testRnd
relay-log = mysql-relay-bin
relay-log = slave-relay.log
relay-log-index = mysql-relay-bin.index
log-error = mysql.err
master-info-file = mysql-master.info
relay-log-info-file = mysql-relay-log.info
log-bin = mysql-bin


After saving this changes  restart mysql service

#service mysql restart
Stopping MySQL:                                            [  OK  ]
Starting MySQL:                                            [  OK  ]


On master 2 (192.168.11.221):
-----------------------------

On master 2 make changes in /usr/local/mysql/etc/my.cnf configuration:


server-id = 2

replicate-do-db = testRnd

log-bin = mysql-bin.log
log-bin-index = master-log-bin.index

binlog-do-db = testRnd

relay-log = mysqld-relay-bin
relay-log = slave-relay.log
relay-log-index = slave-relay-log.index


After saving this changes  restart mysql service

#service mysql restart
Stopping MySQL:                                            [  OK  ]
Starting MySQL:                                            [  OK  ]

STEP:3
==============
Slaves Configuration:

Enter your MySQL shell:
#mysql -u root -p

Unlock your db tables with read lock on Master 1 server:

mysql>USE testRnd;
mysql>UNLOCK TABLES;
mysql>FLUSH TABLES WITH READ LOCK;

In this read lock condition we need to dump the testRnd DB using !!another console!! from Master 1 and initiate below command to find out "MASTER_LOG_FILE" and "MASTER_LOG_POS" values

mysqldump -p testRnd > testRnd_28_12_2021.sql

or 

mysqldump -u root -p --all-databases --master-data > dbdump.sql (To backup all DB)

Copy dump sql file to 55 (Slave) server:

scp testRnd_28_12_2021.sql 192.168.11.221:/tmp

mysql> show master status \G
*************************** 1. row ***************************
            File: mysql-bin.000001
        Position: 107
    Binlog_Do_DB: testRnd
Binlog_Ignore_DB:
1 row in set (0.00 sec)

Than exit from the console to release the read lock.

Setup slave replication on master 2 (192.168.11.221):

We need to restore the dumped DB "testRnd_28_12_2021.sql" which we got from Master 1.

mysql -p testRnd < testRnd_28_12_2021.sql

mysql>USE testRnd;
mysql>UNLOCK TABLES;
mysql>FLUSH TABLES WITH READ LOCK;
mysql> exit;
mysql> mysql -p use testRnd
mysql> STOP SLAVE;
mysql> CHANGE MASTER TO MASTER_HOST = '192.168.11.222', MASTER_USER = 'rep', MASTER_PASSWORD = 'rep123', MASTER_LOG_FILE = 'mysql-bin.000009', MASTER_LOG_POS=154;

Now, start your replication on server:
mysql> START SLAVE;

Check your slave status and your configuration:
mysql>SHOW SLAVE STATUS\G



Setup slave replication on master 1 (192.168.11.222):

mysql>USE testRnd;
mysql>UNLOCK TABLES;
mysql>FLUSH TABLES WITH READ LOCK;
mysql> CHANGE MASTER TO MASTER_HOST = '192.168.11.221', MASTER_USER = 'rep', MASTER_PASSWORD = 'rep123', MASTER_LOG_FILE = 'mysql-bin.000009', MASTER_LOG_POS=154;

Here MASTER_LOG_FILE = 'mysql-bin.000001', MASTER_LOG_POS=107 these values will be found from Master 2 (192.168.11.221) mysql console command "show master status \G"

mysql> START SLAVE;


Check your slave status and your configuration:
mysql>SHOW SLAVE STATUS\G




If Replication Brek:
====================

Unlock your db tables with read lock on Primay Server (172.16.172.16):

mysql -p
mysql>USE cc;
mysql>UNLOCK TABLES;
mysql>FLUSH TABLES WITH READ LOCK;

In this read lock condition we need to dump the cc DB using !!another console!! from Primary server and initiate below command to find out "MASTER_LOG_FILE" and "MASTER_LOG_POS" values

mysql> show master status \G
*************************** 1. row ***************************
            File: mysql-bin.000001
        Position: 107
    Binlog_Do_DB: cc,cc
Binlog_Ignore_DB:
1 row in set (0.00 sec)

Than exit from the console to release the read lock.

mysqldump -p testRnd > testRnd_28_12_2021.sql

scp testRnd_28_12_2021.sql 192.168.11.221:/tmp


Setup slave replication on Secondary (172.16.172.17):

mysql –p
drop database testRnd;
create database testRnd;
We need to restore the dumped DB "testRnd_28_12_2021.sql" which we got from Master 1.
mysql -p testRnd < testRnd_28_12_2021.sql

mysql> STOP SLAVE;
mysql> RESET SLAVE;
mysql> CHANGE MASTER TO MASTER_HOST = '192.168.11.222', MASTER_USER = 'rep', MASTER_PASSWORD = 'rep123', MASTER_LOG_FILE = 'mysql-bin.000003', MASTER_LOG_POS=1783;


mysql> START SLAVE;
mysql> show slave status\G;



====================================================================

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.172.16
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000025
          Read_Master_Log_Pos: 12518
               Relay_Log_File: gtalkpbx-02-relay-bin.000006
                Relay_Log_Pos: 4578
        Relay_Master_Log_File: mysql-bin.000025
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 12518
              Relay_Log_Space: 4740
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
1 row in set (0.00 sec)







