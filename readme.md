# 在docker中测试redis5集群

### 相关说明

1、配置了8个redis实例，分别为node1-node8  
其中node1-node6为三主三从，node7、node8用于后续新增/删除节点测试

2、Redis5不再需要`redis-trib.rb`，直接用`redis-cli`命令即可完成搭建

3、本文只修改集群必不可少的配置，其他与集群没直接关系的配置不做修改，例如：`appendonly`、 `requirepass`等

4、redis-cluster不支持docker的`--link`，即`bridge`模式的network，而docker for windows又不支持host模式，所以本文配置将所有端口映射到物理机，通过物理机进行配置

5、示例物理机IP是`192.168.1.108`，根据需要修改即可

### 修改配置文件

```
# 关闭保护模式，需要从物理机IP组成集群
protected-mode no
# 修改端口号，注意因为是单机配置，每个container的端口号都得不一样
port 7001
# 启用集群
cluster-enabled yes
```

> 其他集群参数（例如：`cluster-config-file`、 `cluster-node-timeout`）都不是必须要改的


### 启动容器

#### 1、通过`docker-compose up -d`启动相关容器

#### 2、创建三主三从集群

将node1-6构成三主三从的集群，随便找一个节点，执行如下命令

```shell
docker-compose exec node6 redis-cli --cluster-replicas 1 --cluster create 192.168.1.108:7001 192.168.1.108:7002 192.168.1.108:7003 192.168.1.108:7004 192.168.1.108:7005 192.168.1.108:7006
```

> 物理机IP是`192.168.1.108`，这个看具体情况修改
> `--cluster-replicas 1`表示每个主节点有一个从节点

返回示例
```shell
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.1.108:7005 to 192.168.1.108:7001
Adding replica 192.168.1.108:7006 to 192.168.1.108:7002
Adding replica 192.168.1.108:7004 to 192.168.1.108:7003
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: ab11d16be02d01d000c56c93970e31335d16928b 192.168.1.108:7001
   slots:[0-5460] (5461 slots) master
M: 3621d0d2c0b8a1e8dd1dde5d7b0b57ca20878f10 192.168.1.108:7002
   slots:[5461-10922] (5462 slots) master
M: 2eccdfd4f36e3907acad2c2fe8eb1cf1c2c628a4 192.168.1.108:7003
   slots:[10923-16383] (5461 slots) master
S: 967d6c698531e39ef2f3c7fbbcedcde7afa38d92 192.168.1.108:7004
   replicates 3621d0d2c0b8a1e8dd1dde5d7b0b57ca20878f10
S: b8e5dbb59e652a86e6a74468029943c592a9b114 192.168.1.108:7005
   replicates 2eccdfd4f36e3907acad2c2fe8eb1cf1c2c628a4
S: 3a948fe130b4e458853f3925c8e6c88f1a951216 192.168.1.108:7006
   replicates ab11d16be02d01d000c56c93970e31335d16928b
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
....
>>> Performing Cluster Check (using node 192.168.1.108:7001)
M: ab11d16be02d01d000c56c93970e31335d16928b 192.168.1.108:7001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: b8e5dbb59e652a86e6a74468029943c592a9b114 172.21.0.1:7005
   slots: (0 slots) slave
   replicates 2eccdfd4f36e3907acad2c2fe8eb1cf1c2c628a4
S: 967d6c698531e39ef2f3c7fbbcedcde7afa38d92 172.21.0.1:7004
   slots: (0 slots) slave
   replicates 3621d0d2c0b8a1e8dd1dde5d7b0b57ca20878f10
M: 2eccdfd4f36e3907acad2c2fe8eb1cf1c2c628a4 172.21.0.1:7003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: 3621d0d2c0b8a1e8dd1dde5d7b0b57ca20878f10 172.21.0.1:7002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 3a948fe130b4e458853f3925c8e6c88f1a951216 172.21.0.1:7006
   slots: (0 slots) slave
   replicates ab11d16be02d01d000c56c93970e31335d16928b
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

这里有个需要手动确认的地方  
Can I set the above configuration? (type 'yes' to accept): **yes**  
输个`yes`就好

至此，集群就配置好了，关闭某个节点模拟宕机很简单，随便玩就好了

#### 3、新增节点（扩容）

命令格式

redis-cli --cluster add-node `<新节点IP:新节点端口>` `<任意一个集群节点IP:端口>`

```shell
docker-compose exec node6 redis-cli --cluster add-node 192.168.1.108:7007 192.168.1.108:7001
```

通过`cluster nodes`命令可以看到，现在有4个master了，但是新节点没有槽位（`slots`），需要为其分配槽位

#### 4、为新增主节点分配槽位

命令格式

redis-cli --cluster reshard `<任意一个集群节点IP:端口>`

```shell
docker-compose exec node6 redis-cli --cluster reshard 192.168.1.108:7001
```

返回示例
```
>>> Performing Cluster Check (using node 192.168.1.108:7001)
M: ab11d16be02d01d000c56c93970e31335d16928b 192.168.1.108:7001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: b8e5dbb59e652a86e6a74468029943c592a9b114 172.21.0.1:7005
   slots: (0 slots) slave
   replicates 2eccdfd4f36e3907acad2c2fe8eb1cf1c2c628a4
M: 7a3d7fe734370070c8dff44befb3d711cade8556 172.21.0.1:7007
   slots: (0 slots) master
S: 967d6c698531e39ef2f3c7fbbcedcde7afa38d92 172.21.0.1:7004
   slots: (0 slots) slave
   replicates 3621d0d2c0b8a1e8dd1dde5d7b0b57ca20878f10
M: 2eccdfd4f36e3907acad2c2fe8eb1cf1c2c628a4 172.21.0.1:7003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: 3621d0d2c0b8a1e8dd1dde5d7b0b57ca20878f10 172.21.0.1:7002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 3a948fe130b4e458853f3925c8e6c88f1a951216 172.21.0.1:7006
   slots: (0 slots) slave
   replicates ab11d16be02d01d000c56c93970e31335d16928b
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 3000
What is the receiving node ID? 7a3d7fe734370070c8dff44befb3d711cade8556
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all
```

此处有3个地方需要输入

1、为新节点分配多少个槽位，例如：3000
```shell
How many slots do you want to move (from 1 to 16384)? 3000
```

2、接收槽位的节点ID，此处为：7a3d7fe734370070c8dff44befb3d711cade8556（即新增主节点的ID）
```shell
What is the receiving node ID? 7a3d7fe734370070c8dff44befb3d711cade8556
```

3、从哪些节点挪槽位过来，`all`为随机从其他节点分配，也可以手动指定节点，这里设为`all`
```shell
Source node #1: all
```

回车后系统会将槽位重新分配，通过`cluster nodes`命令可以查看新的槽位分配情况


#### 4、新增从节点

新增从节点有两种方式，都很简单

4.1、可以直接在加节点的时候指定

命令格式

redis-cli --cluster add-node `<新节点IP:新节点端口>` `<任意一个集群节点IP:端口>` --cluster-slave --cluster-master-id `<对应主节点ID>`

```shell
docker-compose exec node6 redis-cli --cluster add-node 192.168.1.108:7008 192.168.1.108:7001 --cluster-slave --cluster-master-id 7a3d7fe734370070c8dff44befb3d711cade8556
```

4.2、也可以先往集群新增节点，然后在cli命令中指定

1、新增节点操作参考第三节

2、设置对应主节点
```shell
docker run -it --rm redis redis-cli -p 7008 -h 192.168.1.108 -c
cluster replicate 7a3d7fe734370070c8dff44befb3d711cade8556
```


#### 5、删除从节点

直接删就好了

命令格式

redis-cli --cluster del-node `<任意一个集群节点IP:端口>` `<需要删除的节点ID>`

```shell
docker-compose exec node6 redis-cli --cluster del-node 192.168.1.108:7001 1c7bc488c32c844ce804daedeaa12e3350c099c8
```

> 被移除的节点会关闭

#### 6、删除主节点

删除主节点比较复杂，直接删的话会报节点非空错误，需要先将槽位全部转移到集群内的其他节点，最后删除主节点

6.1 移除槽位（跟新增的时候一样，都是重新分配槽位）
```shell
docker-compose exec node6 redis-cli --cluster reshard 192.168.1.108:7001
```

返回示例

```
>>> Performing Cluster Check (using node 192.168.1.108:7001)
M: ab11d16be02d01d000c56c93970e31335d16928b 192.168.1.108:7001
   slots:[999-5460] (4462 slots) master
   1 additional replica(s)
S: b8e5dbb59e652a86e6a74468029943c592a9b114 172.21.0.1:7005
   slots: (0 slots) slave
   replicates 2eccdfd4f36e3907acad2c2fe8eb1cf1c2c628a4
M: 7a3d7fe734370070c8dff44befb3d711cade8556 172.21.0.1:7007
   slots:[0-998],[5461-6461],[10923-11921] (2999 slots) master
S: 967d6c698531e39ef2f3c7fbbcedcde7afa38d92 172.21.0.1:7004
   slots: (0 slots) slave
   replicates 3621d0d2c0b8a1e8dd1dde5d7b0b57ca20878f10
M: 2eccdfd4f36e3907acad2c2fe8eb1cf1c2c628a4 172.21.0.1:7003
   slots:[11922-16383] (4462 slots) master
   1 additional replica(s)
M: 3621d0d2c0b8a1e8dd1dde5d7b0b57ca20878f10 172.21.0.1:7002
   slots:[6462-10922] (4461 slots) master
   1 additional replica(s)
S: 3a948fe130b4e458853f3925c8e6c88f1a951216 172.21.0.1:7006
   slots: (0 slots) slave
   replicates ab11d16be02d01d000c56c93970e31335d16928b
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 3000
What is the receiving node ID? ab11d16be02d01d000c56c93970e31335d16928b
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 7a3d7fe734370070c8dff44befb3d711cade8556
Source node #2: done
```

此处有3个地方需要输入

1、把7007的槽位移走，全部移走的话，直接输入一个大于等于槽数的数字即可（其实输个10000也可以）
```shell
How many slots do you want to move (from 1 to 16384)? 3000
```

2、接收槽位的节点ID，这里偷懒全部移到同一个节点，理论上也可以分多次转移给多个节点，保存平均
```shell
What is the receiving node ID? ab11d16be02d01d000c56c93970e31335d16928b
```

3、从7007节点移出
```shell
Source node #1: 7a3d7fe734370070c8dff44befb3d711cade8556
Source node #2: done
```

6.2 删除节点
```shell
docker-compose exec node6 redis-cli --cluster del-node 192.168.1.108:7001 7a3d7fe734370070c8dff44befb3d711cade8556
```
