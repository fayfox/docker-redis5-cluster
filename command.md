# Redis5集群相关的常用命令

### 创建集群

redis-cli --cluster-replicas 1 --cluster create 192.168.1.108:7001 192.168.1.108:7002 192.168.1.108:7003 192.168.1.108:7004 192.168.1.108:7005 192.168.1.108:7006

> `--cluster-replicas 1`：每个主节点有一个从节点

### 通过redis-cli连接一个集群

```shell
# 直接新起一个容器连接
docker run -it --rm redis redis-cli -p 7006 -h 192.168.1.108 -c

# 利用现有容器的redis-cli客户端连接
docker-compose exec node6 redis-cli -p 7006 -h 192.168.1.108 -c
```

> `-c`表示以集群的方式连接，如果没有这个参数，则为普通连接，当获取不在当前节点的数据时，会触发一个moved异常

### 查看集群概况

此命令需要先用redis-cli连上集群
```shell
192.168.1.108:7006> cluster info
```

### 查看集群节点情况

此命令需要先用redis-cli连上集群
```shell
192.168.1.108:7006> cluster nodes
```

### 检查集群情况

redis-cli --cluster check `<任意一个集群节点IP:端口>`

```shell
docker-compose exec node6 redis-cli --cluster check 192.168.1.108:7001
```

> 感觉得到的信息比`cluster info`, `cluster nodes`更详细，且不需要先进入集群

### 向集群新增节点

redis-cli --cluster add-node `<新节点IP:新节点端口>` `<任意一个集群节点IP:端口>`

新增节点直接指定为从节点

redis-cli --cluster add-node `<新节点IP:新节点端口>` `<任意一个集群节点IP:端口>` --cluster-slave --cluster-master-id `<对应主节点ID>`

```shell
# 新增节点
docker-compose exec node6 redis-cli --cluster add-node 192.168.1.108:7007 192.168.1.108:7001

# 新增节点的同时设置该新增节点为某个主节点的从节点
docker-compose exec node6 redis-cli --cluster add-node 192.168.1.108:7008 192.168.1.108:7001 --cluster-slave --cluster-master-id 7a3d7fe734370070c8dff44befb3d711cade8556
```

### 重新分配槽位（slot）

redis-cli --cluster reshard `<任意一个集群节点IP:端口>`

```shell
docker-compose exec node6 redis-cli --cluster reshard 192.168.1.108:7001
```

### 从集群删除节点

redis-cli --cluster del-node `<任意一个集群节点IP:端口>` `<需要删除的节点ID>`

```shell
docker-compose exec node6 redis-cli --cluster del-node 192.168.1.108:7001 1c7bc488c32c844ce804daedeaa12e3350c099c8
```

### 重新平衡槽位

redis-cli --cluster rebalance `<任意一个集群节点IP:端口>`

```shell
docker-compose exec node6 redis-cli --cluster rebalance 192.168.1.108:7002
```

> `rebalance`操作后所有节点的槽位会相同（可能相差1），但是并不一定会连续

### 批量处理

redis-cli --cluster call `<任意一个集群节点IP:端口>` command arg arg .. arg

```shell
docker-compose exec node6 redis-cli --cluster call 192.168.1.108:7001 dbsize
```

# 与普通单机操作不用的地方

### get/set等操作
会被重定向到对应slot节点

### keys操作
只能列出当前节点的数据