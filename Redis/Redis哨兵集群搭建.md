## 1、背景

由于业务需要重新搭建一个一主二从三哨兵的Redis哨兵集群。

先简单说明下Redis三种集群模式，以及其中哨兵模式的特点。

Redis具有以四种模式，除了单点模式外其他三种是集群模式

* 单点模式，单个节点，挂了就GG了
* 主从模式，一个主节点以及一个以上从节点构成，每个从节点都有主节点所有数据的备份，主节点可读写，从节点只读。一般搭配读写分离使用，主节点只写。主节点挂了之后无法使用，做不到高可用。

* 哨兵模式，主从模式的升级版，除了主从节点外，还怎加了哨兵节点，哨兵节点负责监控数据节点的状态，当主节点宕机是负责故障转移，将一个从节点转化为新的主节点。
* cluster模式，容易混淆的名字，会将数据按key的hash值区分道不同槽位并进而分片，每片数据会存储到一个主节点及其从节点。比如三主三从，数据分作三份存到三个主节点，从节点则备份对应主节点的数据。当某个主节点挂掉时，集群会将其从节点选为新的主节点从而实现高可用。这模式可以解决单个服务器容量有限的问题。

哨兵模式适合只有中小型数据不多的项目，它有以下特性：

* 除了主从节点这些数据节点外，有单独的哨兵节点，哨兵节点通过各节点包括哨兵节点的消息队列订阅和发送消息来和监控和控制数据节点。
* 当主节点异常比如宕机或网络故障时，哨兵在设定时间内无法收到消息就会认定其主观下线，当认同主节点的哨兵数目超过设定值（一般超过半数）时主节点就变成客观下线
* 确认主节点客观下线后哨兵将会选举一个哨并节点来主持主张转移，一般最早发现主节点下线的哨兵
* 主持故障转移的哨兵会从从节点选出一个最优的作为新的主节点，一般是数据最新的从节点
* 主持哨兵通知该从节点，从节点确认后正式升级为主节点，主持哨兵通知各个节点修改信息，更新配置，最后结束故障转移流程
* 哨兵节点应该为单数，否则不好设定故障转移的触发哨兵数，容易导致脑裂



## 2、部署环境

机器 三台服务器 10.xxx.xxx.135、10.xxx.xxx.136、10.xxx.xxx.137

centos 7.9系统

搭建一主二从+三哨兵的哨兵集群，每个机器上一个Redis节点+一个哨兵节点

## 3、版本选择

目前线上使用的是版本6.06，而6.2更新了一些控制指令功能，比如 SENTINEL CONFIG GET、SENTINEL CONFIG SET两个指令以便不停机情况下热更配置，这对哨兵模式下Redis集群的线上维护十分重要。

由于6.2版本官方推荐最新的6.2.7，所以基于此版本进行部署

## 4、Redis的持久化模式选择

目前有RDB、AOF、混合模式，一般来说使用混合模式最好，综合了RDB和AOF的优缺点。

不过网上资料对如何配置混合模式比较多错误的说明。开启混合模式最关键的三个配置是下面三个

```
#关闭RDB持久化模式
save ""
#开启AOF模式
appendonly yes
#开启混合持久化模式
aof-use-rdb-preamble yes
```
因为混合持久化基于AOF模式，AOF指令压缩时在AOF文件前面写全量压缩数据，不需要单独的RDB文件。所以设置混合模式开关 aof-use-rdb-preamble 为yes（5.0后的版本默认yes）同时必须要设置AOF模式appendonly为yes。且最好关闭RDB模式即设置 save ""，否则会持久化多一份数据到单独的dump.rdb文件。

使用上面的配置后，到Redis修改几个数据，使用 bgrewriteaof 指令触发AOF重写，再写几个数据。再到数据目录cat appendonly.aof就可以看到前面是压缩数据，后面是Redis指令集的混合数据。 

## 5、服务器参数优化计基础软件安装

修改优化部分系统参数，需要sudo权限

修改overcommit_memory参数,放宽内存分配条件，避免Reids持久化时失败

修改net.core.somaxconn 、net.ipv4.tcp_max_syn_backlog 参数增加tcp连接数
vim /etc/sysctl.conf  加入以下参数
```
vm.overcommit_memory=1
net.core.somaxconn=2048
net.ipv4.tcp_max_syn_backlog=5000
```

执行以下命令，使配置生效。
sysctl -p

加大最大文件句柄数到10032
```
ulimit -Sn 10032
```

安装工具程序，部分程序可能有了

```
yum -y install cpp
yum -y install binutils
yum -y install glibc
yum -y install glibc-kernheaders
yum -y install glibc-common
yum -y install glibc-devel
yum -y install gcc
yum -y install make
yum -y install tcl
```

##  6、搭建Redis

如果后期打算使用非root用户启动redis以下指令必须切回非root用户

新建需要用到的文件夹
mkdir -p /app/redis
mkdir -p /app/redis/data/7000
mkdir /app/redis/logs/

安装redis，这里会安装redis-server redis-cli redis-benchmark三个到/app/redis/redis-6.2.7的bin目录
```
cd /app/redis
wget  http://download.redis.io/releases/redis-6.2.7.tar.gz
tar vxf redis-6.2.7.tar.gz
cd  /app/redis/redis-6.2.7
make
make PREFIX=/app/redis/redis-6.2.7 install
```

给终端程序加软链接
```
sudo ln -s /app/redis/redis-6.2.7/bin/redis-cli /usr/bin/redis-cli
```

#将配置文件cp一份到外部目录修改

```
cd /app/redis
cp /app/redis/redis-6.2.7/redis.conf /app/redis/redis.conf
```

默认配置大部分比较合理了，按下面的值使用vim修改部分配置并增加重命名危险指令配置即可，maxmemory值扣掉1-2GB给系统用设满即可
```
bind 0.0.0.0 
protected-mode no
port 7000 
tcp-backlog 1022
daemonize yes 
#关闭RDB持久化模式，使用混合持久化模式，混合持久化基于AOF模式，AOF指令压缩时在AOF文件前面写全量压缩数据，不需要单独的RDB文件
save ""
#开启AOF模式
appendonly yes
#开启混合持久化模式，5.0后的版本默认已开启
aof-use-rdb-preamble yes
dir "/app/redis/data/7000"
maxmemory 6000000000
pidfile "/var/run/redis_7000.pid"
logfile "/app/redis/logs/redis_7000.log"
#配置从节点指向的主节点，主节点不需要配置这个
replicaof 10.xxx.xxx.135 7000  
#rename dangerous command, make it hard to use
rename-command FLUSHALL joYAPNXRPmcarcR4
rename-command FLUSHDB  ""
rename-command KEYS     ""
rename-command DEBUG    ""
```

vim 编写start.sh脚本 
```
vim /app/redis/start_redis.sh
```
写入以下内容
```
/app/redis/redis-6.2.7/bin/redis-server /app/redis/redis.conf
tail -f /app/redis/logs/redis_7000.log
```

给脚本加权限
```
chmod +x start_redis.sh
```

使用启动指令启动
```
./start_redis.sh
```

使用以下指令查看数据
```
redis-cli -h 127.0.0.1 -p 7000 
```

Redis还有四个关键配置，这次没有使用，以后搭建集群可能会用到，放在下面：

以下两个配置，如果业务项目使用Redis仅用于缓存就不需要，因为不太担心一致性，可用性更重要。如果数据是长期有用的，就需要配置避免一致性问题
当可同步副本少于1时，主节点不可用，此配置时为了避免异常的master仍然修改输入数据导致数据不一致且故障恢复后也无法恢复。
```
min-replicas-to-write 1
```

配置超过10s副本无通讯就表示联系中断
```
min-replicas-max-lag 10
```

由于以前旧的哨兵集群没有密码，为了方便迁移，本次搭建也不设置密码，如果要设置自身密码和从节点的同步密码，注意所有Redis主从节点要设置一样的密码增加以下两个配置即可

```
requirepass "xxxxx" 
masterauth "xxxxx"
```

另外注意很多文章会建议用下面的配置重命名 CONFIG 指令，但是哨兵集群故障转移时需要用到config指令，不能重命名
```
rename-command CONFIG   ruiwkfkalFJLKJKkjKFKJK
```

## 7、搭建哨兵节点

将配置文件cp一份到外部目录修改
```
cd /app/redis
cp /app/redis/redis-6.2.7/sentinel.conf /app/redis/
```

大部分默认的哨兵节问题已经比较完善，修改以下配置即可
```
port 27000
##关闭保护模式，可以外部访问。 
protected-mode:no 
##设置为后台启动。 
daemonize:yes 
##日志文件
logfile "/app/redis/logs/sentinel.log"
##指定主机IP地址和端口，并且指定当有2台哨兵认为主机挂了，则对主机进行容灾切换。 
sentinel monitor mymaster 10.xxx.xxx.135 7000 2
##这里设置了主机多少秒无响应，则认为挂了。 
sentinel down-after-milliseconds mymaster 10000
##主备切换时，最多有多少个slave同时对新的master进行同步，这里设置为默认的1。 
sentinel parallel-syncs mymaster 1
##故障转移的超时时间，这里设置为30S 
sentinel failover-timeout mymaster 30000

```

编写启动脚本 
```
vim /app/redis/start_sentinel.sh
```
写入以下内容
```
/app/redis/redis-6.2.7/bin/redis-sentinel /app/redis/sentinel.conf
tail -f /app/redis/logs/sentinel.log
```

给脚本加权限
```
chmod +x start_sentinel.sh
```

使用启动指令启动
```
./start_sentinel.sh
```

哨兵都启动后，可使用如下命令查看哨兵信息

```
redis-cli -p 27000 
info sentinel 
```

目前哨兵节点没有使用密码，如果需要使用密码可以增加以下两个配置
设置监管的Redis的密码
```
sentinel auth-pass <master-group-name> <password>
```

设置自身密码，注意所有哨兵节点使用一样的密码
```
requirepass "your_password_here"
```

哨兵节点完全启动后会在其配置文件末尾记录目前的集群状态，写在 # Generated by CONFIG REWRITE 注释后。

到这里哨兵集群就搭建完毕了，注意要清空测试数据请参考 11、部分特殊情况处理 章节的清空测试操作。

##  8、功能验证及测试

主要检查以下几项内容

### 哨兵节点配置自检

使用下方指令让哨兵节点自检配置是否正常
```
redis-cli -p 27000 
SENTINEL CKQUORUM mymaster
```

得到如下结果,就表示正常
```
OK 3 usable Sentinels. Quorum and failover authorization can be reached
```

### 数据同步测试
在主节点修改数据到从节点检查数据主从数据是否同步，即在主节点修改数据后，检查从节点的数据是否跟随变化

在135主节点用set指令修改数据
```
redis-cli -p 7000 
set a a999
```

在136从节点用get指令获取数据对比，发现同步正常
```
redis-cli -p 7000 
get a
```

### 故障转移测试

使用kill -9 杀掉主节点进程和同服务器上的哨兵节点,稍等15秒后进入哨兵节点查询主节点是否已发生变化

```
redis-cli -p 27000 
SENTINEL get-master-addr-by-name mymaster
```

可看到以下结果，新的主节点已经被选出来了

```
1) "10.xxx.xxx.137"
2) "7000"
```

关闭主节点到选出新主节点期间，如果查看哨兵节点的日志，可以看到故障转移日志。

```
99695:X 16 Jun 2022 14:16:15.952 # +sdown master mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:15.952 # +sdown sentinel aea0ad04ade01a7e779caa9acb80263a849b883e 10.xxx.xxx.135 27000 @ mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:16.053 # +odown master mymaster 10.xxx.xxx.135 7000 #quorum 2/2
99695:X 16 Jun 2022 14:16:16.053 # +new-epoch 1
99695:X 16 Jun 2022 14:16:16.053 # +try-failover master mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:16.070 # +vote-for-leader 30595219ee83bcdbe8f5f06054e9d990c6de3c7a 1
99695:X 16 Jun 2022 14:16:16.098 # e893d2288f67eaa86d71e228d5575d8f036d71cc voted for 30595219ee83bcdbe8f5f06054e9d990c6de3c7a 1
99695:X 16 Jun 2022 14:16:16.123 # +elected-leader master mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:16.123 # +failover-state-select-slave master mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:16.182 # +selected-slave slave 10.xxx.xxx.137:7000 10.xxx.xxx.137 7000 @ mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:16.182 * +failover-state-send-slaveof-noone slave 10.xxx.xxx.137:7000 10.xxx.xxx.137 7000 @ mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:16.283 * +failover-state-wait-promotion slave 10.xxx.xxx.137:7000 10.xxx.xxx.137 7000 @ mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:16.737 # +promoted-slave slave 10.xxx.xxx.137:7000 10.xxx.xxx.137 7000 @ mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:16.737 # +failover-state-reconf-slaves master mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:16.750 * +slave-reconf-sent slave 10.xxx.xxx.136:7000 10.xxx.xxx.136 7000 @ mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:17.175 # -odown master mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:17.708 * +slave-reconf-inprog slave 10.xxx.xxx.136:7000 10.xxx.xxx.136 7000 @ mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:17.709 * +slave-reconf-done slave 10.xxx.xxx.136:7000 10.xxx.xxx.136 7000 @ mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:17.771 # +failover-end master mymaster 10.xxx.xxx.135 7000
99695:X 16 Jun 2022 14:16:17.771 # +switch-master mymaster 10.xxx.xxx.135 7000 10.xxx.xxx.137 7000
99695:X 16 Jun 2022 14:16:17.771 * +slave slave 10.xxx.xxx.136:7000 10.xxx.xxx.136 7000 @ mymaster 10.xxx.xxx.137 7000
99695:X 16 Jun 2022 14:16:17.771 * +slave slave 10.xxx.xxx.135:7000 10.xxx.xxx.135 7000 @ mymaster 10.xxx.xxx.137 7000
99695:X 16 Jun 2022 14:16:27.794 # +sdown slave 10.xxx.xxx.135:7000 10.xxx.xxx.135 7000 @ mymaster 10.xxx.xxx.137 7000
```

简单解释下如下：
* +sdown 表明发现节点异常
* +odown 表明多个哨兵节点判断节点异常已经符合客观下线条件，下线的是主节点，就触发了选新主的流程。
* +new-epoch 表示生成一个版本号来标识这次选主行为
* +try-failover 表示故障转移流程开始
* +vote-for-leader 表示选举除了主持此次流程的哨兵节点，其id可以和哨兵节点配置文件尾部被哨兵进程附加写进去的id对比了解具体是哪个节点
* +elected-leader 只是一个说明，说明要从集群里选出新主
* +failover-state-select-slave 也是声明，表示开始要从从节点中选出新主节点
* +selected-slave 表示选出了从节点137作为新主
* +failover-state-send-slaveof-noone 表示发了通知给137从节点告知其被选为新主节点了
* +failover-state-wait-promotion 表示等待新主节点回复
* +promoted-slave 表示从节点137已经答复，同意成为新主节点
* 后续的日志基本意思是更新各个节点的配置，将135这个旧主节点转换角色为从节点，不做一一解释，到 failover-end 时整个故障转移流程算是正式结束

故障转移成功后，所有Redis节点包括哨兵节点都会修改其配置。

原来没有 replicaof 配置的旧主节点启动后会在配置文件末尾会添加以下内容，表明自身变成了137的从节点
```
# Generated by CONFIG REWRITE
user default on nopass ~* &* +@all
replicaof 10.xxx.xxx.137 7000
```

新主节点则会被移除掉 replicaof 配置，然后在配置文件末尾会被添加以下内容
```
# Generated by CONFIG REWRITE
user default on nopass ~* &* +@all
```

其他从节点则会被修改配置文件中的 replicaof 配置，改为指向新主节点IP

而哨兵节点则会修改监控配置,指向新的主节点
```
sentinel monitor mymaster 10.xxx.xxx.137 7000 2
```

### redis性能测试
使用官方的性能测试工具来进行，执行下面的指令就是对redis服务测试每次数据长度为400字节时，20个连接做100000次get、set操作时的性能表现
```
cd /app/redis/redis-6.2.7/bin
./redis-benchmark -h ip -a password -p 7000 -c 20 -t get,set -d 400 -n 100000 -q
```

整个集群搭建完毕后压测结果如下
```
SET: 30129.56 requests per second, p50=0.559 msec                   
GET: 33624.75 requests per second, p50=0.487 msec 
```

## 9、搭建好的集群情况

* Redis数据节点、哨兵节点都是 10.xxx.xxx.135、 10.xxx.xxx.136、10.xxx.xxx.137 三个机器上各有一个，可以按 常用指令章节的指令 查看当前主节点
* 使用端口： Redis主从节点 7000，哨兵节点 27000
* Redis程序目录 /app/redis/redis-6.2.7/bin
* Redis数据目录 /app/redis/data/7000
* 启动脚本位置 /app/redis 下的 start_redis.sh  start_sentinel.sh
* 配置文件位置 /app/redis 下的 redis.conf  sentinel.conf
* 运行日志目录 /app/redis/logs


程序使用Redis哨兵节点配置：
```
redis.sentinel.1=10.xxx.xxx.135:27000
redis.sentinel.2=10.xxx.xxx.136:27000
redis.sentinel.3=10.xxx.xxx.137:27000
```

哨兵模式的机制要有多数个哨兵节点存活才可以进行故障转移选出新主，且只有主节点可以写数数据，从节点只能读数据。

这次搭建的集群在一个设备宕机或网络异常时可以在10s左右选出新主恢复正常。

如果出现两台服务器同时宕机或者网络异常的情况时，且唯一正常机器是从节点则集群无法正常使用，需要紧急响应处理，尽快恢复。


## 10、常用指令

查看哨兵信息
```
redis-cli -p 27000 
info sentinel 
```

哨兵节点自检配置是否正常
```
redis-cli -p 27000 
SENTINEL CKQUORUM mymaster
```

在哨兵节点查看当前主节点
```
redis-cli -p 27000 
SENTINEL get-master-addr-by-name mymaster
```

##  11、部分特殊情况处理

**如果要将机器克隆来做新的集群，要如何清理历史数据避免误操作关联到旧集群？**
* 清空数据文件夹 /app/redis/data
* 注释掉 redis.conf、sentinel.conf 文件末尾自动写入的配置也即是 # Generated by CONFIG REWRITE 注释后的部分。但如果运行期间使用过CONFIG指令修改过部分配置的，需要保留这部分热更的配置
* 修改sentinel.conf文件里 sentinel monitor 配置的IP为xxxx
* 修改redis.conf文件里replicaof部分的配置的IP为xxxx

**故障转移后不想将旧主节点接回集群中，如果处理？**
* 哨兵集群保存有各个节点的信息，即使已经下线，可以使用指令 SENTINEL RESET mastername 来强制刷新节点列表为目前有效节点的数据，mastername就是集群名字，比如我们这里的集群名字 mymaster。 

**如果出现特殊情况故障转移流程不起作用**
* 使用指令 SENTINEL FAILOVER <master name> 强制触发一次故障转移流程，此时不需要其他哨兵节点同意。

**如何清空测试数据**
* 用Rename后的FLUSHALL指令清空数据，FLUSHALL不会触发刷盘
* bgrewriteaof 触发混合模式刷盘
* scan 0 验证是否还有数据

**keys指令被禁用后如何在不知道有什么key的情况查看数据**
* 使用scan指令，scan 0 MATCH * COUNT 20 TYPE STRING ，第一个参数是游标值，如果想看更多数据，可以使用输出结果里第一个数据数字作为下次查询的游标值

```
redis 127.0.0.1:7000> scan 0 MATCH * COUNT 10 TYPE STRING
1) "17"
2)  1) "key:12"
    2) "key:8"
    3) "key:4"
    4) "key:14"
    5) "key:16"
    6) "key:17"
    7) "key:15"
    8) "key:10"
    9) "key:3"
   10) "key:7"
redis 127.0.0.1:7000> scan 17 MATCH * COUNT 10 TYPE STRING
1) "0"
2) 1) "key:5"
   2) "key:18"
   3) "key:0"
   4) "key:2"
   5) "key:19"
   6) "key:13"
   7) "key:6"
   8) "key:9"
   9) "key:11"
```

**宕机的服务器或者异常关闭的节点如何处理**
* 重启异常的服务器或重启异常的Redis节点、哨兵节点即可，哨兵集群的故障转移机制会处理好节点的数据同步和主从身份转换



##  13、参考资料

[哨兵集群官方说明文档](https://redis.io/docs/manual/sentinel/)

[Redis的Linux系统优化](https://cachecloud.github.io/2017/02/16/Redis%E7%9A%84Linux%E7%B3%BB%E7%BB%9F%E4%BC%98%E5%8C%96/)

[阿里云帮助文档，优化连接数设置](https://help.aliyun.com/document_detail/174950.html)

[哨兵集群故障转移失效处理](http://blog.itpub.net/30393770/viewspace-2706857/)

