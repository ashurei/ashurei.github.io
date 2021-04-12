---
title: "[PostgreSQL] Streaming Replication (12.5)"
description: "PostgreSQL Streaming Replication"
last_modified_at:
date: 2021-04-12
location: content
---

# [PostgreSQL] Streaming Replication

* PostgreSQL version : 12.5
* OS : CentOS 7.9
* Service Network : 192.168.56.0/24
* Private Network : 10.0.0.0/24
* node#1 : 192.168.56.11, 10.0.0.1
* node#2 : 192.168.56.12, 10.0.0.2


## 1. Create test data
---
``` sql
postgres=# create database db01;
postgres=# create user u1;
postgres=# \c db01 u1
db01=> create table t1 (a1 integer, a2 varchar(20));
db01=>insert into t1 values (1, 'test');
```


## 2. Configure postgresql.conf
---
``` bash
listen_addresses = '*'
port=5444
logging_collector = on
archive_mode = on 
archive_command = 'test ! -f /PG/ARCH/%f && cp %p /PG/ARCH/%f'
```


## 3. Configure pg_hba.conf
---
``` bash
[postgres]$ vi $PGDATA/pg_hba.conf
host    replication     all             192.168.56.0/24         md5
host    replication     all             10.0.0.0/24             md5

[postgres]$ pg_ctl reload
```


## 4. pg_basebackup
---
* [node#1]
``` sql
postgres=# create user repl password 'repl' replication;
```
* [node#2]
``` bash
[postgres]$ pg_basebackup -h 192.168.56.11 -p 5444 -D /PG/DATA -U repl -R -X stream -P
```
* -R or --write-recovery-conf : standby.signal 파일 생성 및 postgresql.auto.conf 파일 구성
* -X stream or -Xs : streaming replication 방식
* -P : progress 표시

> _PostgreSQL 12 부터 recovery.conf 는 사라지고 postgresql.auto.conf 와 standby.signal 로 변경되었다._

## 5. Check standby (node#2)
---
``` sql
[postgres]$ psql db01 u1
db01=> select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t
(1 row)

db01=> insert into t1 values (4,'asdf');
ERROR:  cannot execute INSERT in a read-only transaction
```

## ※ 참고 문서
> * *<https://www.postgresql.org/docs/12/runtime-config-replication.html>*
> * *<https://www.postgresql.org/docs/12/app-pgbasebackup.html>*
> * *<https://www.postgresql.org/docs/12/release-12.html>*
