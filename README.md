# mysqlsync
## MySQL Sync :: Replication Table from Oracle to MySQL Asynchronously

### Installing

1. Download platform specific binaries:
```sh 
$ sudo wget https://github.com/ojengwa/mysqlsync/raw/master/scripts/linux/mysqlsync_linux64_10204.bin -o /usr/local/mysqlsync.bin
```
Or for Windows users (you need to download curl first):
```sh
c:\ curl https://github.com/ojengwa/mysqlsync/raw/master/scripts/nt/mysqlsync.exe -o mysqlsync.exe
```

### Usage Instruction

There are lot’s of products for replication data from Oracle to MySQL based on redo log file, such as SharePlex and GoldenGate. I will instroduce another utility named “mysqlsync” which can replicate data from Oracle to MySQL based on the materialized view log for change data capture.

First, let’s create a testing table and the materialized view log in Oracle.
```sql
SQL> create table t_mysql (id number(10) not null primary key, col2 int);

Table created.

SQL> create materialized view log on t_mysql with primary key, sequence;

Materialized view log created.
```
It’s better to create an index on the “SEQUENCE$$” column of the materialized view log table as following.
```sql
SQL> desc MLOG$_T_MYSQL
 Name                      Null?    Type
 ---------------------------------- ----------------

 ID                                 NUMBER(10)
 SEQUENCE$$                         NUMBER
 SNAPTIME$$                         DATE
 DMLTYPE$$                          VARCHAR2(1)
 OLD_NEW$$                          VARCHAR2(1)
 CHANGE_VECTOR$$                    RAW(255)
 XID$$                              NUMBER

SQL> create index id_mlog_t_mysql on mlog$_t_mysql (sequence$$);
```

Now create the table on MySQL instance with same table structure.
```sql
mysql> create table t_mysql (id int not null primary key, col2 int);
Query OK, 0 rows affected (0.02 sec)
```

A configuration file is required for “mysqlsync” utility to specify the tables need to be replicated and their primary key columns and the materialized view log tables.
```sh
$ cat sync.cfg
# Source # Key Col # Log Table     # Target
 t_mysql # id      # mlog$_t_mysql # t_mysql
 ```
 
 Then star the “mysqlsync” utility with the above configuration file.
 
 ```sh
 $ mysqlsync user1=system/oracle user2=test/test@db01:3306:test config=sync.cfg
 ```
The replication is working now, let’s insert some rows into Oracle database.
```sql
SQL> insert into t_mysql values (1, 10);
1 row created.
SQL> commit;
Commit complete.
SQL> insert into t_mysql values (2, 10);
1 row created.
SQL> commit;
Commit complete.
```
Don’t forget to issue the `commit` command for Oracle, after the commit command, the replication is working with the following message.

```sh
2016-03-27 16:58:22 -- Replicate from T_MYSQL to t_mysql,   1 messages processed.
2016-03-27 16:58:46 -- Replicate from T_MYSQL to t_mysql,   1 messages processed.
```
Let’s insert more rows now.
```sql
SQL> insert into t_mysql select object_id, data_object_id
  2     from all_objects where object_id > 2 and object_id < 10;
7 rows created.
SQL> commit;
Commit complete.
```
You can query the rows from MySQL database now.
```sql
mysql> select * from t_mysql;
+----+------+
| id | col2 |
+----+------+
|  1 |   10 |
|  2 |   10 |
|  3 |    3 |
|  4 |    2 |
|  5 |    2 |
|  6 |    6 |
|  7 |    7 |
|  8 |    8 |
|  9 |    9 |
+----+------+
9 rows in set (0.00 sec
```
