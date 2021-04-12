---
title: "[PostgreSQL] PostgreSQL Automatic Failover"
description: "PostgreSQL Automatic Failover"
last_modified_at:
date: 2021-04-12
location: content
---

# [PostgreSQL] PostgreSQL Automatic Failover

* PostgreSQL version : 12.5
* OS : CentOS 7.9
* Service Network : 192.168.56.0/24
* Private Network : 10.0.0.0/24
* /etc/hosts 정보
``` bash
192.168.56.11   vm01
192.168.56.12   vm02
192.168.56.13   pg_cluster_vip
10.0.0.1        vm01-prv
10.0.0.2        vm02-prv
```

* Pacemaker 를 활용한 PostgreSQL 이중화 구성 가이드 문서.
* PostgreSQL 설치 및 Replication 구성은 아래 링크 참고.
 * {{ site.url }}/docs/postgresql/postgres_install
 * {{ site.url }}/docs/postgresql/postgres_streaming_replication

## 1. Pacemaker 설치
---
``` bash
[root]# yum install -y pacemaker pcsd
[root]# systemctl enable pcsd
```

## 2. PAF 설치
---
``` bash
[root]# yum install -y perl-Data-Dumper
[root]# rpm -ivh resource-agents-paf-2.3.0-1.noarch
```
> PAF 다운로드: <https://github.com/ClusterLabs/PAF/releases>
* paf 설치 후 /usr/lib/ocf/resource.d/heartbeat/pgsqlms 파일 생성 확인


## 3. pg_hba.conf 확인
---
* Replication 망을 Private Network 로 구성하기 위해 pg_hba.conf 의 replication 섹션에 Service/Private Network 추가 확인
``` bash
[node#both]
host    replication     all             192.168.56.0/24         md5
host    replication     all             10.0.0.0/24             md5
```


## 4. Replication 추가 설정
---
* [node#1]
``` sql
postgres=# select pg_create_physical_replication_slot('node01');
postgres=# alter system set primary_conninfo = 'user=repl password=repl host=vm02-prv port=5444 application_name=vm01-prv sslmode=disable sslcompression=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any';
postgres=# alter system set recovery_target_timeline = 'latest';
postgres=# alter system set primary_slot_name = 'node02';
```
* [node#2]
``` sql
postgres=# select pg_create_physical_replication_slot('node02');
postgres=# alter system set primary_conninfo = 'user=repl password=repl host=vm01-prv port=5444 application_name=vm02-prv sslmode=disable sslcompression=0 gssencmode=disable krbsrvname=postgres target_session_attrs=any';
postgres=# alter system set recovery_target_timeline = 'latest';
postgres=# alter system set primary_slot_name = 'node01';
```
* host : 상대 노드 의 IP
* application_name : 자신의 host명
* primary_slot_name 은 임의로 설정하되 상대 노드의 slot name.


## 5. Cluster 구성
---
### 1) Cluster 초기화
* setup 시 Service/Private Network 이중화 구성
``` bash
[root]# pcs cluster stop --all
[root]# pcs cluster destroy --all
[root]# pcs cluster setup --name pg_cluster vm01-prv,vm01 vm02-prv,vm02
[root]# pcs cluster start --all
```
* Private Network 가 우선이 될 수 있도록 먼저 넣어줌. (vm01-prv,vm01 vm02-prv,vm02)

### 2) Cluster ring 설정 확인
``` bash
[root]# corosync-cfgtool -s
Printing ring status.
Local node ID 1
RING ID 0
id = 10.0.0.1
status = ring 0 active with no faults
RING ID 1
id = 192.168.56.11
status = ring 1 active with no faults
```

### 3) Cluster 구성
``` bash
pcs cluster cib pg_cfg

pcs -f pc_cfg property set no-quorum-policy="ignore"
pcs -f pc_cfg property set stonith-enabled="false"
pcs -f pc_cfg resource defaults resource-stickiness=10
pcs -f pc_cfg resource defaults migration-threshold="1"

# VIP
pcs -f pg_cfg resource create VIP ocf:heartbeat:IPaddr2 ip=192.168.56.13 cidr_netmask=25 nic="enps03" op monitor interval=5s

# PostgreSQL
pcs -f pg_cfg resource create PG ocf:heartbeat:pgsqlms \
bindir="/PG/bin" \
pgdata="/PG/DATA" \
pgport="5444" \
op start timeout=60s \
op stop timeout=60s \
op promote timeout=30s \
op demote timeout=30s on-fail="stop" \
op monitor interval=15s timeout=10s role="Master" on-fail="restart" \
op monitor interval=16s timeout=10s role="Slave" on-fail="restart" \
op notify timeout=60s

# Master
pcs -f pg_cfg resource master PG-master PG master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true

# Constraint
pcs -f pg_cfg constraint colocation add VIP with master PG-master INFINITY
pcs -f pg_cfg constraint order promote PG-master then start VIP symmetrical=false kind=Mandatory
pcs -f pg_cfg constraint order demote PG-master then stop VIP symmetrical=false kind=Mandatory

# Ping to gateway
pcs -f pg_cfg resource create ping_check ocf:pacemaker:ping dampen=5s multiplier=1000 host_list="192.168.56.1" clone
pcs -f pg_cfg constraint location VIP rule score=-INFINITY pingd lt 1 or not_defined pingd

pcs cluster cib-push pg_cfg
```
* pg_ctl stop 시 fail-over 를 위해 아래 설정 필요.
(restart 옵션은 기본적으로 다른 노드에서 시도)
``` bash
pcs resource update PG op monitor interval=15s role=Master timeout=10s on-fail="restart"
pcs resource update PG op monitor interval=16s role=Slave timeout=10s on-fail="restart"
```
* demote 된 후 stop 상태로 유지하기 위해 아래 설정 필요
(restart 옵션은 기본적으로 다른 노드에서 시도. 이것이 실패할 경우 fail되었던 node 에서 재시작됨)
``` bash
pcs resource update PG op demote interval=0s timeout=30s on-fail="stop"
```


## 5. pgsqlms 수정
---
``` bash
[root]# vi /usr/lib/ocf/resource.d/heartbeat/pgsqlms
sub _confirm_stopped {
...
    if ( $controldata_rc == $OCF_RUNNING_MASTER ) {
        # The controldata has not been updated to "shutdown".
        # It should mean we had a crash on a primary instance.
        ocf_exit_reason(
            'Instance "%s" controldata indicates a running primary instance, the instance has probably crashed',
            $OCF_RESOURCE_INSTANCE );
        #return $OCF_FAILED_MASTER;
        return $OCF_NOT_RUNNING;
```
* kill postgres 시 pg_controldata $PGDATA|grep state 의 결과가 'in production' 인 상태로 남아 있게 되고,
이 경우 $OCF_FAILED_MASTER 를 return 하면서 fail-over 를 시키지 않고 block 상태로 빠지게 됨.
강제로 fail-over 시키려면 위와 같이 return 값을 $OCF_NOT_RUNNING 으로 변경해주면 fail-over 가 가능함.
대신 상황에 따라 old primary 는 재구성해야 할 가능성이 있으므로 fail-over 시 확인 필요.


## 6. Check script
---
* [master]
``` bash
(insert.sh)
psql db01 u1 -c "insert into t1 values (111, 'test')"
psql db01 u1 -c "select * from t1"
psql db01 u1 -c "select pg_current_wal_lsn()"
```
* [master&slave]
``` bash
(chkSlave.sh)
psql -c "select * from pg_stat_replication"
psql -c "select pg_is_in_recovery()"
psql -c "select pg_last_wal_receive_lsn(),pg_last_wal_replay_lsn()"
psql db01 u1 -c "select * from t1"
```


## 7. Test scenario
---
* pg_ctl stop
* kill postgres
* ifdown 'Service Network'
* ifdown 'Private Network'


## 8. 관리 명령어
---
* cluster 유지보수 모드 전환. 리소스 관리하지 않음.
``` bash
pcs property set maintenance-mode=true
```
* 개별 resource 에 대해 freeze.
``` bash
pcs resource unmanage <resource id>
pcs resource manage 명령으로 해제
```
* master node 를 standby 로 변환하면 slave 로 fail-over 됨.
``` bash
pcs cluster standby <node명>
pcs resource unstandby <node명> 명령으로 해제
```


## ※ 참고 문서
> * PAF github (최신 버전 v2.3.0)
>> * <https://github.com/ClusterLabs/PAF>
> * PAF 구성 가이드
>> * <http://clusterlabs.github.io/PAF/Quick_Start-CentOS-7.html>
> * PAF 구성 소개 블로그
>> * <https://pgstef.github.io/2018/02/07/introduction_to_postgresql_automatic_failover.html>
> * Mandatory 설정
>> * <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/high_availability_add-on_reference/s1-orderconstraints-haar>
> * on-fail 설정 참고 문서
>> * <https://wiki.clusterlabs.org/wiki/PgSQL_Replicated_Cluster#PostgreSQL>
>> * <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/high_availability_add-on_reference/s1-resourceoperate-HAAR#tb-resource-operation-HAAR>
> * replication slot
>> * <https://www.postgresql.org/docs/12/warm-standby.html>
> * Ping + location 관련 설정
>> * <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/high_availability_add-on_reference/s1-moving_resources_due_to_connectivity_changes-haar>

