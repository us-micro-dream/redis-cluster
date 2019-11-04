# docker-compose部署redis集群

利用docker-compose单节点部署redis集群

# redis集群条件

需要开启两个端口

- 普通客户端通信端口（通常为6379）
- 群集总线端口（客户端端口+ 10000）必须可以从所有其他群集节点访问


普通客户端通信端口 = redis.conf文件中配置的port(默认值是6379)


群集总线端口 = port + 10000

# 环境

- centos7，ip地址为 106.12.76.77
- docker
- docker-compose


# 目录结构

```
 |-- redis-cluster
    |-- 7000
        |-- conf
            |-- redis.conf
        |-- data
        |-- docker-compose.yml
    |-- 7001
        |-- conf
            |-- redis.conf
        |-- data
        |-- docker-compose.yml
    |-- 7002
        |-- conf
            |-- redis.conf
        |-- data
        |-- docker-compose.yml
    |-- 7003
        |-- conf
            |-- redis.conf
        |-- data
        |-- docker-compose.yml
    |-- 7004
        |-- conf
            |-- redis.conf
        |-- data
        |-- docker-compose.yml
    |-- 7005
        |-- conf
            |-- redis.conf
        |-- data
        |-- docker-compose.yml
``` 


# redis.conf 模板

```
port {$POST}
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```


# docker-compose.yml 模板

```
version: '3.1'
services:
  redis:
    container_name: redis7000
    image: redis:5.0.6
    ports:
      - "7000:7000"
      - "17000:17000"       
    volumes:
      - ./conf:/usr/local/etc/redis
      - ./data:/data
    restart: always
    command: redis-server /usr/local/etc/redis/redis.conf
    networks:
      - "redis-net"
networks:
    redis-net:
        external: true
```



# 创建集群

7000 ~ 7005的容器都启动之后，随便进入一个容器

比如
```
docker-compose exec redis bash
```

执行创建集群命令
```
redis-cli --cluster create 物理机ip:7000 物理机ip:7001 物理机ip:7002 物理机ip:7003 物理机ip:7004 物理机ip:7005 --cluster-replicas 1

```


该选项--cluster-replicas 1意味着我们要为每个创建的主机都希望有一个从机


结果

```
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 106.12.76.73:7004 to 106.12.76.73:7000
Adding replica 106.12.76.73:7005 to 106.12.76.73:7001
Adding replica 106.12.76.73:7003 to 106.12.76.73:7002
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 3b99def34ff7323132a3df4b63f4d8d175c726a6 106.12.76.73:7000
   slots:[0-5460] (5461 slots) master
M: 02cec662386fde3d35117a26db4986139029e25d 106.12.76.73:7001
   slots:[5461-10922] (5462 slots) master
M: eb7a5958f87c88a9ff1d098fd04c4816174eece5 106.12.76.73:7002
   slots:[10923-16383] (5461 slots) master
S: cf9dcaa1768f63dda646fb3f2cf5f69bee5d0850 106.12.76.73:7003
   replicates 3b99def34ff7323132a3df4b63f4d8d175c726a6
S: a593b2444a4c2685cf427c0f89723669cd2cd37d 106.12.76.73:7004
   replicates 02cec662386fde3d35117a26db4986139029e25d
S: 07c46b594efd0fa7da4711ee333e8cd4189acf26 106.12.76.73:7005
   replicates eb7a5958f87c88a9ff1d098fd04c4816174eece5
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 106.12.76.73:7000)
M: 3b99def34ff7323132a3df4b63f4d8d175c726a6 106.12.76.73:7000
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: a593b2444a4c2685cf427c0f89723669cd2cd37d 106.12.76.73:7004
   slots: (0 slots) slave
   replicates 02cec662386fde3d35117a26db4986139029e25d
S: cf9dcaa1768f63dda646fb3f2cf5f69bee5d0850 106.12.76.73:7003
   slots: (0 slots) slave
   replicates 3b99def34ff7323132a3df4b63f4d8d175c726a6
S: 07c46b594efd0fa7da4711ee333e8cd4189acf26 106.12.76.73:7005
   slots: (0 slots) slave
   replicates eb7a5958f87c88a9ff1d098fd04c4816174eece5
M: eb7a5958f87c88a9ff1d098fd04c4816174eece5 106.12.76.73:7002
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: 02cec662386fde3d35117a26db4986139029e25d 106.12.76.73:7001
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

```

## 关于master和slave的选择

命令
```

# ip都是外网，可以访问的

redis-cli --cluster create 106.12.76.73:7000 106.12.76.73:7001 \
106.12.76.73:7002 106.12.76.73:7003 106.12.76.73:7004 106.12.76.73:7005 \
--cluster-replicas 1
```

这个命令，  ip总数 / 2 ，前面的会自动设置为master，后面的是master对应的slave?
    
    
还有一点，创建的集群感觉会，自动分配平均的***哈希槽***
    
    
# 添加节点

## 添加一个master节点

### 创建一个新的节点，并加入集群

在单机环境下，可以通过添加一个 7006的目录结构，对应的修改redis.conf和docker-compose.yml的配置信息

***注意***，新创建的 data目录要空啊，你不要用cp命令，把data里面的数据也复制了
redis.conf
```
port 7006
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes

```

docker-compose.yml
```
version: '3.1'
services:
  redis:
    container_name: redis7006
    image: redis:5.0.6
    ports:
      - "7006:7006"
      - "17006:17006"       
    volumes:
      - ./conf/redis.conf:/usr/local/etc/redis/redis.conf 
      - ./data:/data
    restart: always
    command: redis-server /usr/local/etc/redis/redis.conf    
    networks:
      - "redis-net"
networks:
    redis-net:
        external: true

```

启动该节点，总结；一个reids.conf配置文件启动一个redis进程，就是一个redis节点，用docker去启动是因为，让它们模拟redis单独的在每个服务器上


启动完之后，说明该7006节点已经加入了集群，加入集群是加入集群，但7006节点在集群中并没有启动分担的工作。

进入7006容器里面，执行命令，会看到 7006 content后面是没有参数的。说明 7006 节点只是加入了集群，没有分配***哈希槽！！***

***查看集群命令***。 进入容器执行的命令 -p 端口值，要和该redis对应的实例tcp两个端口的那第一个，即（7006提供服务的端口） 否则会出现什么127.0.0.1 700*的错误。但这不影响redis集群的部署和运行，redis的集群和部署都采用了外网的方式，上面的创建的集群都是外网。
```
redis-cli -p 7006 cluster nodes
```

输出的结果是我复制官网的，我的那个屏幕输出太多，找不到了，这些不重要。我们关心的重点是
***myself*** 那里就行，***connected*** 后面是空的，说明没有分配***哈希槽***，具体解释去[官网](https://redis.io/topics/cluster-tutorial)看吧
```
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385543178575 0 connected 5960-10921
3fc783611028b1707fd65345e763befb36454d73 127.0.0.1:7004 slave 3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 0 1385543179583 0 connected
f093c80dde814da99c5cf72a7dd01590792b783b :0 myself,master - 0 0 0 connected
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385543178072 3 connected
a211e242fc6b22a9427fed61285e85892fa04e08 127.0.0.1:7003 slave 97a3a64667477371c4479320d683e4c8db5858b1 0 1385543178575 0 connected
97a3a64667477371c4479320d683e4c8db5858b1 127.0.0.1:7000 master - 0 1385543179080 0 connected 0-5959 10922-11422
3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7005 master - 0 1385543177568 3 connected 11423-16383
```

### 为新的master节点，分配哈希槽

1. 随便进入一个redis实例，这里我进入7000这个实例。如果你当前在7006，端口换成7006就行。
```
redis-cli --cluster reshard 106.12.76.73:7000
```

执行命令后，会出现
```
How many slots do you want to move (from 1 to 16384)?
```

哈希槽个人总结点:
- redis集群中，总共就只有16384个位置
- 一个redis master占多少个哈希槽是可以自己设置的。
- 哈希槽可重新分配
- redis-cli -p 7006 cluster nodes 命令connected后面就是，该redia节点占有多少个哈希槽


2. 哈希槽选择分配

我不知道是根据什么方案分配的哈希槽，我田了一个最大值 16384
```
16384
```

3. 选择哈希槽接收的node节点

命令 cluster nodes ，可以展示出 nodes节点的id
```
redis-cli -p 7006 cluster nodes
```

输入接收哈希槽的节点
```
How many slots do you want to move (from 1 to 16384)? 16384
What is the receiving node ID? 3b99def34ff7323132a3df4b63f4d8d175c726a6
```

选择哪个节点的数据，需要重新进入哈希槽分配。这部分不是很懂，我填了 all
```
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all
```

最后结果的是，我原来master节点 7000 、7001、7002的数据全部没有了，数据全部都移动到了 7006 节点上。跑程序往redis添加数据的时候，只添加到 7006 实例中。查看节点的信息，7000、7001、7002节点***connected***后面没有哈希槽


总结：

redis，目前没有提供会为节点自动分配哈希槽，需要手工配置。


哈希槽平均分配 = 16384 / master节点数

## 添加一个slave从节点

## 添加一组节点；1组 = 1 master + 1 slave

# 删除节点

```
redis-cli --cluster del-node `<ip>`:`<port>` `<node-id>`
```
第一个参数只是集群中的一个随机节点，第二个参数是您要删除的节点的ID。

您也可以用相同的方法删除主节点，但是要删除主节点，它必须为空。***如果主节点不为空，则需要先将数据从其重新分片到所有其他主节点。***

删除主节点的另一种方法是在其一个从节点上对其执行手动故障转移，并在该节点成为新主节点的从节点之后删除该节点。显然，这在您要减少群集中的主节点的实际数量时无济于事，在这种情况下，需要重新分片
