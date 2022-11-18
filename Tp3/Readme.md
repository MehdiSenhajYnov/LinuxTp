# **Module 2 : Réplication de base de données**

**1. Mise en place de la machine "maitre"**
```
[mehdi@db ~]# sudo nano /etc/my.cnf.d/mariadb-server.cnf
[mysqld]
log-bin=mysql-bin
server-id=101
[mehdi@db ~]# systemctl restart mariadb
[mehdi@db ~]# mysql -u root -p
MariaDB [(none)]> grant replication slave on *.* to repl_user@'%' identified by 'password'; 
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> flush privileges; 
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> exit
Bye
```



**2.	On change des parametres dans la machine "esclave"**
```
[mehdi@slavedb ~]# sudo nano /etc/my.cnf.d/mariadb-server.cnf
[mysqld]
log-bin=mysql-bin
server-id=102
read_only=1
report-host=node01.srv.world
[mehdi@slavedb ~]# systemctl restart mariadb
```


















**3. On prepare le backup dans la machine "maitre"**
```
[mehdi@db ~]# mkdir /home/mariadb_backup
[mehdi@db ~]# mariabackup --backup --target-dir /home/mariadb_backup -u root -p password
[00] 2021-08-03 16:14:27 Connecting to MySQL server host: localhost, user: root, password: set, port: not set, socket: not set
[00] 2021-08-03 16:14:27 Using server version 10.3.28-MariaDB
mariabackup based on MariaDB server 10.3.28-MariaDB Linux (x86_64)
[00] 2021-08-03 16:14:27 uses posix_fadvise().
[00] 2021-08-03 16:14:27 cd to /var/lib/mysql/
.....
.....
[00] 2021-08-03 16:14:30         ...done
[00] 2021-08-03 16:14:30 Redo log (from LSN 1640838 to 1640847) was copied.
[00] 2021-08-03 16:14:30 completed OK!
```



























**4. On passe le backup de la machine "maitre" à celle "esclave" avec sftp**
   **On regle les parametres de replication**
```
[mehdi@slavedb ~]# sudo systemctl stop mariadb
[mehdi@slavedb ~]# sudo rm -rf /var/lib/mysql/*

[mehdi@slavedb ~]# sudo mariabackup --prepare --target-dir mariadb_backup
mariabackup based on MariaDB server 10.3.28-MariaDB Linux (x86_64)
[00] 2021-08-03 16:23:30 cd to mariadb_backup/
[00] 2021-08-03 16:23:30 open files limit requested 0, set to 1024
.....
.....
2021-08-03 16:23:30 0 [Note] InnoDB: Starting crash recovery from checkpoint LSN=1640838
[00] 2021-08-03 16:23:30 Last binlog file , position 0
[00] 2021-08-03 16:23:30 completed OK!

[mehdi@slavedb ~]# sudo mariabackup --copy-back --target-dir mariadb_backup
mariabackup based on MariaDB server 10.3.28-MariaDB Linux (x86_64)
[01] 2021-08-03 16:23:46 Copying ibdata1 to /var/lib/mysql/ibdata1
[01] 2021-08-03 16:23:46         ...done
.....
.....
[01] 2021-08-03 16:23:46 Copying ./xtrabackup_info to /var/lib/mysql/xtrabackup_info
[01] 2021-08-03 16:23:46         ...done
[00] 2021-08-03 16:23:46 completed OK!

[mehdi@slavedb ~]# sudo chown -R mysql. /var/lib/mysql
[mehdi@slavedb ~]# sudo systemctl start mariadb
[mehdi@slavedb ~]# sudo cat mariadb_backup/xtrabackup_binlog_info
mysql-bin.000003        642     0-101-2

[mehdi@slavedb ~]# sudo mysql -u root -p

MariaDB [(none)]> change master to 
master_host='10.0.0.31',
master_user='repl_user',
master_password='password',
master_log_file='mysql-bin.000001',
master_log_pos=642;
Query OK, 0 rows affected (0.191 sec)

MariaDB [(none)]> start slave; 
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> show slave status\G 
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 10.0.0.31
                   Master_User: repl_user
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: mysql-bin.000001
           Read_Master_Log_Pos: 642
                Relay_Log_File: mariadb-relay-bin.000002
                 Relay_Log_Pos: 555
         Relay_Master_Log_File: mysql-bin.000001
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
           Exec_Master_Log_Pos: 642
               Relay_Log_Space: 866
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
              Master_Server_Id: 101
                Master_SSL_Crl:
            Master_SSL_Crlpath:
                    Using_Gtid: No
                   Gtid_IO_Pos:
       Replicate_Do_Domain_Ids:
   Replicate_Ignore_Domain_Ids:
                 Parallel_Mode: conservative
                     SQL_Delay: 0
           SQL_Remaining_Delay: NULL
       Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
              Slave_DDL_Groups: 0
Slave_Non_Transactional_Groups: 0
    Slave_Transactional_Groups: 0
1 row in set (0.000 sec)
```