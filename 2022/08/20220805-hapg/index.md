# Postgres 高可用部署验证


## 构架图

参考官方截图

![](/images/arch-single-standby.svg)

## 部署步骤

废话就懒得说了直接上命令

### 2. 准备工作

```bash
#### 1. 内部互通网络
docker network create --driver bridge mynet  --subnet 172.20.0.0/24 --gateway 172.20.0.1
```

### 2. 第一步创建 monitor

```bash
docker run --net mynet -it -d --name pgmonitor-service \
  --ip 172.20.0.20 \
  -v /data/pgmonitor:/data/pgaf \
  -e "PG_AUTOCTL_DEBUG=1" \
  -e "PGDATA=/data/pgaf" \
  hapg:14 pg_autoctl create monitor --no-ssl --auth trust --run
```

运行完以后可以`docker exec pgmonitor-service pg_autoctl show state`命令查看是否已经启动完成
```bash
Name |  Node |  Host:Port |  TLI: LSN |   Connection |      Reported State |      Assigned State
-----+-------+------------+-----------+--------------+---------------------+--------------------
```

## 2. 第二步创建 主节点

```bash
docker run --net mynet -it -d --name pg1-service \
  --ip 172.20.0.30 \
  -v /data/pg1:/data \
  -e "PG_AUTOCTL_DEBUG=1" \
  -e "PGDATA=/data/pgaf" \
  hapg:14 pg_autoctl create postgres --no-ssl --auth scram-sha-256 \
  --hostname pg1-service --pg-hba-lan --username zuser --dbname zdb \
  --monitor "postgresql://autoctl_node@pgmonitor-service/pg_auto_failover" --run
```
等待完成（可以继续使用上述`docker exec pgmonitor-service pg_autoctl show state`查看状态命令），完成大概如下

```bash
  Name |  Node |        Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+------------------+----------------+--------------+---------------------+--------------------
node_1 |     1 | pg1-service:5432 |   1: 0/17598E0 |   read-write |              single |              single
```

然后配置一下主从节点件通信密码

```bash
docker exec pg1-service pg_autoctl config set replication.password h4ckm3m0r3
```

> **注意这里有个坑**，上面的官方文档命令完成以后没有真正创建密码，如果这个时候直接加载启动从节点则会出现如下报错：

```log
08:28:05 9 INFO  Reloading pg_autoctl postgres service [-1]
08:28:05 10 ERROR Connection to database failed: connection to server at "pg1-service" (172.20.0.30), port 5432 failed: FATAL:  password authentication failed for user "pgautofailover_replicator"
08:28:05 10 ERROR Failed to connect to "postgres://pgautofailover_replicator@pg1-service:5432/zdb?" after 16 attempts in 1438 ms, pg_autoctl stops retrying now
08:28:05 10 WARN  Failed to contact the primary node 1 "node_1" (pg1-service:5432)
08:28:06 10 WARN  Failed to connect to "postgres://pgautofailover_replicator@pg1-service:5432/zdb?", retrying until the server is ready
08:28:06 10 ERROR Connection to database failed: connection to server at "pg1-service" (172.20.0.30), port 5432 failed: FATAL:  password authentication failed for user "pgautofailover_replicator"
```

这里需要登录主库创建出密码 命令如下：

```bash
docker exec pg1-service psql postgres -c "alter user pgautofailover_replicator password 'h4ckm3m0r3';"
```

## 第二步创建 创建从节点

```bash
docker run --net mynet -it -d --name pg2-service \
  --ip 172.20.0.40 \
  -v /data/pg2:/data \
  -e "PG_AUTOCTL_DEBUG=1" \
  -e "PGPASSWORD=h4ckm3m0r3" \
  -e "PGDATA=/data/pgaf" \
  hapg:14 pg_autoctl create postgres --no-ssl --auth scram-sha-256 \
  --hostname pg2-service --pg-hba-lan --username zuser --dbname zdb \
  --monitor "postgresql://autoctl_node@pgmonitor-service/pg_auto_failover" --run

docker exec pg2-service pg_autoctl config set replication.password h4ckm3m0r3
# 这里不需要再次创建密码了因为连接成功以后会自动同步过来
```

## 检查最终状态

```bash
# docker exec pgmonitor-service pg_autoctl show state
  Name |  Node |        Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+------------------+----------------+--------------+---------------------+--------------------
node_1 |     1 | pg1-service:5432 |   1: 0/3337FE8 | read-write ! |             primary |      demote_timeout
node_2 |     2 | pg2-service:5432 |   1: 0/3338078 |    read-only |   prepare_promotion |    stop_replication

```

## 检查主备功能

这里我们先去另外一个地方导出数据库备份

```bash 
docker exec dev-pg pg_dump -F c -f /test.dmp -C -E UTF8 -U postgres test_db
docker cp dev-pg:/test.dmp ./
```
导入到主数据库

```bash
# 数据库导入主节点
docker cp test.dmp pg1-service:/
docker exec dev-pg pg_restore --no-owner --role=zuser -d zdb test.dmp
```

登录从数据库查看,可以看到数据已经有了

```bash
chinaddos@liuyou-iMac pgtest % docker exec -it pg2-service psql postgres -d zdb
docker@21c6c8007d75:/$ psql postgres
psql (14.4 (Debian 14.4-1.pgdg100+1))
Type "help" for help.
zdb=# \dt
               List of relations
 Schema |         Name          | Type  | Owner 
--------+-----------------------+-------+-------
 public | test111111111         | table | zuser
 public | test1111111112222     | table | zuser
# ...
```

使用命令停止主库 `docker stop pg1-service`

```log
  Name |  Node |        Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+------------------+----------------+--------------+---------------------+--------------------
node_1 |     1 | pg1-service:5432 |   1: 0/3337FE8 | read-write ! |             primary |      demote_timeout
node_2 |     2 | pg2-service:5432 |   2: 0/333BD10 |    read-only |    stop_replication |    stop_replication

```

直接修改第二个数据库

```sql
zdb=# select * from zw_widgets;
 widget_id |     title     | widget_type 
-----------+---------------+------------
 143786240 | 策略管理  |         1
 143786496 | 策略管理2 |         1
(2 rows)
-- 其他命令
zdb=# update zw_widgets set widget_type = 2;
-- 其他命令
```

再次启动第一个数据库，可以看到主备已经互换了

```
  Name |  Node |        Host:Port |       TLI: LSN |   Connection |      Reported State |      Assigned State
-------+-------+------------------+----------------+--------------+---------------------+--------------------
node_1 |     1 | pg1-service:5432 |   2: 0/5000110 |    read-only |           secondary |           secondary
node_2 |     2 | pg2-service:5432 |   2: 0/5000110 |   read-write |             primary |             primary
```

检查pg1，数据库数据已经同步过来

```sql
zdb=# select * from zw_widgets;
 widget_id |     title     | widget_type 
-----------+---------------+------------
 143786240 | 策略管理  |         2
 143786496 | 策略管理2 |         2
(2 rows)
```

## 补充

1. 怎么调整监测时间相关参数

```
psql -p [port] -d pg_auto_failover
# List the health_check variables
SELECT name, setting FROM pg_settings WHERE name ~ 'pgautofailover\.';
# Check status of database every 5s, setthe timeout to 2s and set the node unhealthy timeout to 5s
ALTER SYSTEM SET pgautofailover.health_check_period TO 5000;
ALTER SYSTEM SET pgautofailover.health_check_timeout TO 2000;
ALTER SYSTEM SET pgautofailover.node_considered_unhealthy_timeout TO 5000;
# Reload config:
select pg_reload_conf();

```

2. pg-hba.conf 配置文件可能会有坑，需要主要自行添加数据库访问权限


## 参考资料

<https://pg-auto-failover.readthedocs.io/en/master/security.html?highlight=password#authentication-with-passwords>

<https://cloud.tencent.com/developer/article/1663944>

<https://github.com/citusdata/pg_auto_failover/blob/master/docker-compose.yml>

