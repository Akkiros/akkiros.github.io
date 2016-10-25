---
layout: post
title: MariaDB Failover 적용
---

실제 웹서비스를 운영하면서 가장 무서운 일중에 하나가 DB서버가 죽는일이 발생하는 것입니다.  
READ용 Slave DB가 죽는다면 그나마 피해가 덜 하지만, WRITE용  Master DB가 죽는다면 실제로 서비스에 큰 문제가 발생하게 됩니다.  
이런 상황을 최대한 문제없이 넘기기 위해서 MariaDB에 Failover를 설정해본 내용을 공유합니다.

## 1. Failover를 지원하는 툴 찾기

Failover를 지원하는 여러가지 툴들이 있습니다. 여기서는 여러가지 툴들에 대한 특징들을 알아봅니다.

### 1. [mysqlfailover](https://dev.mysql.com/doc/mysql-utilities/1.5/en/mysqlfailover.html)

- MySQL Utility에서 제공하는 mysql용 failover 툴입니다.
- 이 툴은 MySQL의 GTID(Global Transaction Identifiers)을 기반으로 동작합니다.
- 상태 체크(health check)는 Ping을 사용합니다.
- 이 툴을 사용하려면 /etc/mysql/my.cnf에 `gtid_mode=ON` 옵션을 사용해야 하는데, MariaDB에서는 해당 옵션이 없어서 mysqlfailover를 MariaDB Failover시 사용할 수 없습니다.
- [MySQL과 MariaDB System Variable 차이점](https://mariadb.com/kb/en/mariadb/system-variable-differences-between-mariadb-100-and-mysql-56/)을 참고해보세요.

### 2. [ProxySQL](http://www.proxysql.com/)

- Query Routing, Query Caching, Statistics 등 쿼리에 관련된 여러가지 작업들을 해주는 툴입니다.
- Master, Slave 관계에 관해 hostgroup라는 개념으로 나누는데, 특정 쿼리 패턴에 따라 원하는 hostgroup에 속한 서버로 쿼리를 보내는 처리를 해줍니다.
- 상태 체크는 별도의 모니터링 모듈을 이용합니다.
- Failover를 지원하는 것 같지만, 아직은 지원하지 않습니다.
- [ProxySQL의 Schema, health check에 관한 설명](http://severalnines.com/blog/mysql-load-balancing-proxysql-overview)을 참고해보세요.

### 3. [MHA(Master High Availability)](https://code.google.com/p/mysql-master-ha/)

- ProxySQL과는 달리 failover에 관련된 기능만을 수행하는 툴입니다.
- 모니터링 서버에는 manager, node를 설치하고, 각 DB(Master, Slave)에는 node들을 설치합니다.
- ssh, select, insert 방식으로 ping 체크를 합니다.
- MariaDB에도 적용할 수 있기 때문에 이 툴을 사용했습니다.

**가장 적합한 MHA를 사용하기로 결정!**

## 2. Replication 설정하기

failover를 하기전에 Master-Slave Replication을 설정해야 합니다.

테스트 환경은 AWS를 이용하였습니다.  
AWS를 사용한다면 하나의 서버에 mariadb를 설치한 후 AMI로 만들어서 나머지 서버들을 만들면 편리합니다.

```
mariadb-server-10.1로 구성된 서버 3대

Master  -  Slave_1
        -  Slave_2
```

**AWS상에서 테스트시 설정해야하는 사항**

- 각 MariaDB를 remote로 접속하기 때문에 `/etc/mysql/my.cnf`에서 `bind-address = 127.0.0.1`으로 되어있는 부분들을 주석처리해야 합니다.
- Scurity Group Inbound에서 3306번 포트와 22번 포트를 열어줘야 합니다. 이 [링크](http://pyrasis.com/book/TheArtOfAmazonWebServices/Chapter05)를 참고하세요.

### 2.1 Master 설정

`sudo vi /etc/mysql/my.cnf`

```
# The following can be used as easy to replay backup logs or forreplication.
# note: if you are setting up a replication slave, see README.Debian about
#       other settings you may need to change.
#server-id               = 1
#report_host            = master1
#auto_increment_increment = 2
#auto_increment_offset  = 1
log_bin                 = /var/log/mysql/mariadb-bin
log_bin_index           = /var/log/mysql/mariadb-bin.index
# not fab for performance, but safer
#sync_binlog            = 1
expire_logs_days        = 10
max_binlog_size         = 100M
# slaves
#relay_log              = /var/log/mysql/relay-bin
#relay_log_index        = /var/log/mysql/relay-bin.index
#relay_log_info_file    = /var/log/mysql/relay-bin.info
#log_slave_updates
#read_only
```

이렇게 나와있는 부분이 있는데(server-id를 검색하면 찾기 쉽다), 해당 부분에서 server-id에 되어있는 주석을 해제합니다.

해당 부분에서 사용하는 옵션들을 보면 아래와 같습니다.

```
server-id               = 1
log_bin                 = /var/log/mysql/mariadb-bin
log_bin_index           = /var/log/mysql/mariadb-bin.index
expire_logs_days        = 10
max_binlog_size         = 100M
```

**server-id?**

> server-id는 replication 상태에서 master와 slave를 구분해주는 값입니다. server-id는 유니크한 값이어야합니다.

파일을 저장하고, mariadb를 재시작해줍니다. `sudo service mysql restart`

그리고 마스터의 상태를 확인하기 위해서 아래 명령어를 실행합니다.

```
MariaDB [(none)]> SHOW MASTER STATUS\G;
*************************** 1. row ***************************
            File: mariadb-bin.000003
        Position: 623
    Binlog_Do_DB:
Binlog_Ignore_DB:
```

여기서 나온 File과 Position은 아래에 slave 설정을 할 때 사용할 값이니 어딘가에 저장해둡니다.

### 2.2 Slave 설정

`sudo vi /etc/mysql/my.cnf`

```
# The following can be used as easy to replay backup logs or forreplication.
# note: if you are setting up a replication slave, see README.Debian about
#       other settings you may need to change.
#server-id               = 1
#report_host            = master1
#auto_increment_increment = 2
#auto_increment_offset  = 1
log_bin                 = /var/log/mysql/mariadb-bin
log_bin_index           = /var/log/mysql/mariadb-bin.index
# not fab for performance, but safer
#sync_binlog            = 1
expire_logs_days        = 10
max_binlog_size         = 100M
# slaves
#relay_log              = /var/log/mysql/relay-bin
#relay_log_index        = /var/log/mysql/relay-bin.index
#relay_log_info_file    = /var/log/mysql/relay-bin.info
#log_slave_updates
#read_only
```

Master과 같이 해당 파일들에서 몇가지 옵션을 수정해줘야 합니다.  
위에서와 같이 server-id에 주석을 해제하고, Master와는 다른 server-id로 수정합니다.(server-id는 유니크해야 합니다)  
그리고 아래 `# slave` 이하의 부분들도 주석을 해제해줍니다.

사용하는 옵션들은 다음과 같습니다.

```
server-id               = 2
log_bin                 = /var/log/mysql/mariadb-bin
log_bin_index           = /var/log/mysql/mariadb-bin.index
expire_logs_days        = 10
max_binlog_size         = 100M
relay_log               = /var/log/mysql/relay-bin
relay_log_index = /var/log/mysql/relay-bin.index
relay_log_info_file     = /var/log/mysql/relay-bin.info
log_slave_updates
read_only
```

slave\_1이라면 server-id를 2로 지정하고, slave\_2라면 server-id를 3으로 지정하면 됩니다.

위에서와 같이 파일을 저장하고, mariadb를 재시작해줍니다. `sudo service mysql restart`


재시작한 후에 mariadb console에 접속하여 master 설정을 추가합니다.

`mysql -u root -p`

```
MariaDB [(none)]> CHANGE MASTER TO
MASTER_HOST='아이피',
MASTER_USER='repliUser',
MASTER_PASSWORD='비밀번호',
MASTER_PORT=포트번호,
MASTER_LOG_FILE='Master File',
MASTER_LOG_POS=Master Position,
MASTER_CONNECT_RETRY=10;
  
MariaDB [(none)]> FLUSH PRIVILEGES;
 
MariaDB [(none)]> START SLAVE;
```

해당 부분에서 Master File과 Master Position은 위에 Master Server에서 `show master status\G;` 로 한 결과값으로 출력되는 File, Position을 각각 입력해줍니다.

해당 설정이 끝났다면 `show slave status\G;` 명령어를 실행하여 replication 상태를 확인해볼 수 있습니다.

```
MariaDB [(none)]> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 
                  Master_User: 
                  Master_Port: 
                Connect_Retry: 10
              Master_Log_File: mariadb-bin.000003
          Read_Master_Log_Pos: 623
```

Slave\_IO\_Status가 Waiting for master to send event라면 replication이 잘 적용된 것입니다.

오류가 발생한다면 위에서 설명한 **AWS상에서 테스트시 설정해야하는 사항**을 참고해보세요.

### 2.3 GTID 사용하기

GTID를 적용하기 위해서는 우선 Master DB에서 GTID 사용을 활성화 해야합니다.

`sudo vi /etc/mysql/my.cnf`

```
gtid-domain-id=1
```

해당 옵션을 추가했다면 아래의 명령어로 현재 binlog 파일의 정보를 확인합니다.

```
MariaDB [(none)]> show master status\G;
*************************** 1. row ***************************
            File: mariadb-bin.000007
        Position: 329
    Binlog_Do_DB:
Binlog_Ignore_DB:
```

그리고 같은 쿼리로 GTID Binlog의 위치를 파악합니다.

```
MariaDB [(none)]> select BINLOG_GTID_POS('mariadb-bin.000007', 329);
+--------------------------------------------+
| BINLOG_GTID_POS('mariadb-bin.000007', 329) |
+--------------------------------------------+
| 0-1-470                                    |
+--------------------------------------------+
```

현재 Master DB의 GTID 정보를 확인했으니 해당 정보들을 Slave들에 적용시킵니다.

`Slave_1`

```
MariaDB [(none)]> stop slave;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> SET GLOBAL gtid_slave_pos = '0-1-470';
Query OK, 0 rows affected, 1 warning (0.01 sec)

MariaDB [(none)]> CHANGE MASTER TO master_use_gtid=slave_pos;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> start slave;
Query OK, 0 rows affected (0.02 sec)
```

`Slave_1`과 마찬가지로 `Slave_2`에서도 똑같이 적용합니다.

```
MariaDB [(none)]> stop slave;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> SET GLOBAL gtid_slave_pos = '0-1-470';
Query OK, 0 rows affected, 1 warning (0.01 sec)

MariaDB [(none)]> CHANGE MASTER TO master_use_gtid=slave_pos;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> start slave;
Query OK, 0 rows affected (0.02 sec)
```

적용이 완료 되었다면 `show slave status\G;` 명령어를 이용해 GTID 설정이 잘 되었는지 확인합니다.

```
MariaDB [(none)]> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: xxx.xxx.xxx.xxx
                  Master_User: 
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mariadb-bin.000007
...

                   Using_Gtid: Slave_Pos
                  Gtid_IO_Pos: 0-1-470
      Replicate_Do_Domain_Ids:
  Replicate_Ignore_Domain_Ids:
                Parallel_Mode: conservative
```

위와같이 Using\_Gtid와 Gtid\_IO\_POS 값이 잘 나온다면 적용이 완료된 것입니다.

#### 참고자료

> [Replication 참고자료](http://www.tutorialbook.co.kr/entry/MariaDB-Mysql-%EC%9D%BC%EB%B0%98-Replication-%EB%B3%B5%EC%A0%9C-%ED%95%98%EA%B8%B0)  
> [GTID 적용 참고자료](https://mariadb.com/blog/enabling-gtids-server-replication-mariadb-100)

## 3. MHA 설정하기
 
### 3.1 설치하기


MHA를 설정하기 위해서는 Master-Slave DB 이외에 모니터링 서버를 한대 따로 두어야합니다.  
모니터링 서버에 MHA manager와 MHA node를 설치하고, 각 DB들에 MHA node을 설치합니다.  
manager는 node끼리의 통신을 통해 현재 DB의 상태를 확인할 수 있습니다.
  
Manager 서버는 DB서버와 같이 mariadb-10.1을 설치해두었습니다.
 
[MHA 설치방법](https://code.google.com/p/mysql-master-ha/wiki/Installation)를 참고하면 되고, 구글 코드에서 파일을 받으려면 404 에러가 나와서 manager, node 파일들은 [MariaDB에서 제공하는 다운로드 링크](https://downloads.mariadb.com/MHA/)에서 받았습니다.
 
이 문서에서는`mha4mysql-manager_0.55-0_all.deb`, `mha4mysql-node_0.54-0_all.deb` 버전을 사용하였습니다.
 
### 3.2 configuration file 생성하기
 
`/etc/mha/app.cnf` 파일을 생성합니다.(mha 폴더는 임의로 생성한 폴더)

```
[server default]
# mysql user and password for MHA (should be created on all DB servers)
user=YOUR DB USER
password=YOUR DB PASSWORD
ssh_user=ubuntu
# working directory on the manager
manager_workdir=/home/ubuntu/mha/app
# working directory on MySQL servers
remote_workdir=/home/ubuntu/mha/app

[server1]
hostname=YOUR MASTER DB IP

[server2]
hostname=YOUR SLAVE 1 DB IP
candidate_master=1

[server3]
hostname=YOUR SLAVE 2 DB IP
no_master=1
```

각 옵션들에 대한 설명은 아래와 같습니다.

- user : mariadb에서 사용하는 user
- password : 해당 user의 password
- ssh\_user : 각 서버끼리 접속할때 사용할 ssh\_user(아래에 key 추가하는 부분 참고)
- manager_workdir : manager가 사용할 directory
- remote_workdir : MariaDB가 사용할 directory
- [server] : 각 서버들. [server1]은 master에 대한 정보를 넣어야합니다.
- candidate_master : 1로 설정된다면 failover시 master가 될 수 있는 slave입니다.
- no_master : 1로 설정된다면 해당 slave는 failover시 master로 선정되지 않습니다.

[더 많은 옵션들에 대한 정보](https://code.google.com/p/mysql-master-ha/wiki/Parameters)에 자세히 나와있습니다.

> MariaDB는 5.5에서 10.x로 버전업이 돼서 10.x를 사용하고 있다면 mha를 사용할때 오류가 발생합니다.  
> 해당 문제를 해결하는 방법은 [여기](https://code.google.com/p/mysql-master-ha/issues/detail?id=70)를 참고해서 version 체크하는 부분을 바꿔주면 됩니다.  
> 하지만 해당 해결 방법은 공식적인 방법이 아니므로 오류가 발생할 여지는 있습니다.

### 3.3 ssh key 등록하기

문제가 발생했는지 확인하거나, 각 상태를 변경해야하기 때문에 ssh를 이용합니다.

그러므로 각 서버에서 `ssh-keygen`명령어를 이용하여 ssh key를 만들고, 만들어진 키를 모든 서버(Monitor, Master, Slave\_1, Slave\_2)의 `~/.ssh/authorized_keys`에 추가시켜주어야합니다.

```
ubuntu@ip-xxx-xxx-xxx-xxx:~$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ubuntu/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/ubuntu/.ssh/id_rsa.
Your public key has been saved in /home/ubuntu/.ssh/id_rsa.pub.
The key fingerprint is:
xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx ubuntu@ip-xxx-xxx-xxx-xxx
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
+-----------------+
```

ssh key 설정이 끝났다면 Monitor 서버에서 `masterha-check-ssh --conf /etc/mha/app.conf`를 실행하면 아래와 같이 성공화면을 볼 수 있습니다.

```
ubuntu@ip-xxx-xxx-xxx-xxx:~$ masterha_check_ssh --conf /etc/mha/app.cnf
Tue Oct 11 09:23:00 2016 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Tue Oct 11 09:23:00 2016 - [info] Reading application default configurations from /etc/mha/app.cnf..
Tue Oct 11 09:23:00 2016 - [info] Reading server configurations from /etc/mha/app.cnf..
Tue Oct 11 09:23:00 2016 - [info] Starting SSH connection tests..
Tue Oct 11 09:23:01 2016 - [debug]
Tue Oct 11 09:23:00 2016 - [debug]  Connecting via SSH from ubuntu@master(master:22) to ubuntu@slave1(slave1:22)..
Tue Oct 11 09:23:00 2016 - [debug]   ok.

...

Tue Oct 11 09:23:01 2016 - [debug]  Connecting via SSH from ubuntu@slave2(slave2:22) to ubuntu@master(master:22)..
Tue Oct 11 09:23:02 2016 - [debug]   ok.
Tue Oct 11 09:23:02 2016 - [info] All SSH connection tests passed successfully.
```

### 3.4 replication 테스트 하기

ssh 테스트가 성공적으로 끝났다면, replication도  `masterha_check_repl --conf /etc/mha/app.cnf` 명렁어를 이용해 테스트 해볼 수 있습니다.

```
ubuntu@ip-xxx-xxx-xxx-xxx:~$ masterha_check_repl --conf /etc/mha/app.cnf
Tue Oct 11 09:36:14 2016 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Tue Oct 11 09:36:14 2016 - [info] Reading application default configurations from /etc/mha/app.cnf..
Tue Oct 11 09:36:14 2016 - [info] Reading server configurations from /etc/mha/app.cnf..
Tue Oct 11 09:36:14 2016 - [info] MHA::MasterMonitor version 0.55.
Tue Oct 11 09:36:14 2016 - [info] Dead Servers:
Tue Oct 11 09:36:14 2016 - [info] Alive Servers:
Tue Oct 11 09:36:14 2016 - [info]   master(master:3306)
Tue Oct 11 09:36:14 2016 - [info]   slave_1(slave_1:3306)
Tue Oct 11 09:36:14 2016 - [info]   slave_2(slave_2:3306)
Tue Oct 11 09:36:14 2016 - [info] Alive Slaves:
Tue Oct 11 09:36:14 2016 - [info]   slave_1(slave_1:3306)  Version=10.1.18-MariaDB-1~trusty (oldest major version between slaves) log-bin:enabled
Tue Oct 11 09:36:14 2016 - [info]     Replicating from master(master:3306)
Tue Oct 11 09:36:14 2016 - [info]     Primary candidate for the new Master (candidate_master is set)
Tue Oct 11 09:36:14 2016 - [info]   slave_2(slave_2:3306)  Version=10.1.18-MariaDB-1~trusty (oldest major version between slaves) log-bin:enabled
Tue Oct 11 09:36:14 2016 - [info]     Replicating from master(master:3306)
Tue Oct 11 09:36:14 2016 - [info]     Not candidate for the new Master (no_master is set)
Tue Oct 11 09:36:14 2016 - [info] Current Alive Master: master(master:3306)

...

Tue Oct 11 09:36:18 2016 - [info] Checking replication health on slave_1..
Tue Oct 11 09:36:18 2016 - [info]  ok.
Tue Oct 11 09:36:18 2016 - [info] Checking replication health on slave_2..
Tue Oct 11 09:36:18 2016 - [info]  ok.
Tue Oct 11 09:36:18 2016 - [warning] master_ip_failover_script is not defined.
Tue Oct 11 09:36:18 2016 - [warning] shutdown_script is not defined.
Tue Oct 11 09:36:18 2016 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
```

위와 같은 메세지를 받아본다면 replication 설정도 잘 되어있다는 이야기입니다.  
해당 메세지를 살펴보면 현재 살아있는 서버들에 대한 정보, 현재 Master에 대한 정보, 현재 Slave들에 대한 정보들을 확인할 수 있습니다.

### 3.5 failover 테스트 하기

ssh, replication 둘 다 테스트를 통과했다면 직접 failover를 테스트해봅니다.

`masterha_manager --conf /etc/mha/app.cnf` 명령어를 실행하면 manager을 실행할 수 있습니다.  
해당 manager은 기본적인 ping(ping, select, insert)테스트를 해본 후 오류가 발생할 때까지 기다립니다.

해당 명령어를 실행한 후 master 서버의 DB를 중지시켜보고 failover가 잘 되는지 확인해봅니다.

```
ubuntu@ip-xxx-xxx-xxx-xxx:~$masterha_manager --conf /etc/mha/app.cnf
```

Master DB에서 mysql을 중지시킵니다.

```
#At Master DB
ubuntu@ip-xxx-xxx-xxx-xxx:~$sudo service mysql stop
```

Master DB가 정지되면 Monotor 서버에서 Master의 상태를 확인한 후 failover를 시작합니다.

```
----- Failover Report -----

app: MySQL Master failover master to slave_1 succeeded

Master master is down!

Check MHA Manager logs at ip-xxx-xxx-xxx-xxx for details.

Started automated(non-interactive) failover.
The latest slave slave_1(slave_1:3306) has all relay logs for recovery.
Selected slave_1 as a new master.
slave_1: OK: Applying all logs succeeded.
slave_2: This host has the latest relay log events.
Generating relay diff files from the latest slave succeeded.
slave_2: OK: Applying all logs succeeded. Slave started, replicating from slave_1.
slave_1: Resetting slave info succeeded.
Master failover to slave_1(slave_1:3306) completed successfully.
```

해당 처리가 끝나면 Slave\_1이 새로운 Master가 되고, Slave\_2는 Slave\_1을 새로운 마스터로 인식하여 Slave\_1의 replication이 됩니다.
