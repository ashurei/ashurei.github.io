---
title: "[PostgreSQL] Source Install (12.5)"
description: "PostgreSQL Source Install"
last_modified_at:
date: 2021-04-12
location: content
---

# [PostgreSQL] Source Install

* PostgreSQL version : 12.5
* OS : CentOS 7.9
* Service Network : 192.168.56.0/24
* Private Network : 10.0.0.0/24
* PGHOME=/PG
* PGDATA=/PG/DATA
* PGPORT=5444

## 1. Download
---
* Home: <https://www.postgresql.org/ftp/source/v12.6/>
* FTP: <https://ftp.postgresql.org/pub/source/v12.6/postgresql-12.6.tar.gz>


## 2. Install
---
``` bash
[root]# yum install -y readline-devel
[root]# useradd -u 500 postgres
[root]# chown postgres. /home/postgres/postgresql-12.6.tar.gz
[root]# mkdir -p /PG/DATA
[root]# chown -R postgres. /PG
[root]# su - postgres

[postgres]$ tar xvfz postgresql-12.6.tar.gz
[postgres]$ cd postgresql-12.6
[postgres]$ ./configure --prefix /PG
[postgres]$ make
[postgres]$ make install
[postgres]$ vi ~/.bash_profile
PATH=$PATH:$HOME/.local/bin:$HOME/bin:/PG/bin
export PGDATA=/PG/DATA
export PGPORT=5444
```


## 3. Create Database
---
``` bash
[postgres]$ initdb -D /PG/DATA
```

## 4. Start Database
---
``` bash
[postgres]$ pg_ctl start -D /PG/DATA
[postgres]$ psql
postgres=> alter user postgres password '비밀번호';
```

## ※ 참고 문서
> *<https://dico.me/etc/articles/165/ko>*
