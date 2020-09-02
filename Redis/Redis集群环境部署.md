# 1简介

在 Redis 3.0 之前，使用 哨兵（sentinel）机制来监控各个节点之间的状态。Redis Cluster 是 Redis 的 分布式解决方案，在 3.0 版本正式推出，有效地解决了 Redis 在 分布式 方面的需求。当遇到单机内存、并发、流量等瓶颈时，可以采用 Cluster 架构方案达到负载均衡的目的

# 2 Redis Cluster集群介绍

Redis Cluster 集群模式通常具有 **高可用**、**可扩展性**、**分布式**、**容错** 等特性。Redis Cluster采用的是**虚拟槽分区。**先来解释一下什么是虚拟槽分区

## 2.1 虚拟槽分区介绍

**虚拟槽分区** 巧妙地使用了 **哈希空间**，使用 **分散度良好** 的 **哈希函数** 把所有数据 **映射** 到一个 **固定范围** 的 **整数集合** 中，整数定义为 **槽**（slot）。这个范围一般 **远远大于** 节点数，比如 Redis Cluster 槽范围是 0 ~ 16383。**槽** 是集群内 **数据管理** 和 **迁移** 的 **基本单位**。采用 **大范围槽** 的主要目的是为了方便 **数据拆分** 和 **集群扩展**。每个节点会负责 **一定数量的槽**，如图所示：

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image002.jpg)

当前集群有 5 个节点，每个节点平均大约负责 3276 个 **槽**。由于采用 **高质量** 的 **哈希算法**，每个槽所映射的数据通常比较 **均匀**，将数据平均划分到 5 个节点进行 **数据分区**。Redis Cluster 就是采用 **虚拟槽分区**。

- **节点1**： 包含 0 到 3276 号哈希槽。
- **节点2**：包含 3277 到 6553 号哈希槽。
- **节点3**：包含 6554 到 9830 号哈希槽。
- **节点4**：包含 9831 到 13107 号哈希槽。
- **节点5**：包含 13108 到 16383 号哈希槽。

这种结构很容易 **添加** 或者 **删除** 节点。如果 **增加** 一个节点 6，就需要从节点 1 ~ 5 获得部分 **槽** 分配到节点 6 上。如果想 **移除** 节点 1，需要将节点 1 中的 **槽** 移到节点 2 ~ 5 上，然后将 **没有任何槽** 的节点 1 从集群中 **移除** 即可。

由于从一个节点将 **哈希槽** 移动到另一个节点并不会 **停止服务**，所以无论 **添加删除** 或者 **改变** 某个节点的 **哈希槽的数量** 都不会造成 **集群不可用** 的状态.

## 2.2 Redis数据分区

上面说过，Redis Cluster 采用 **虚拟槽分区**，所有的 **键** 根据 **哈希函数** 映射到 0~16383 整数槽内，计算公式：slot = CRC16（key）& 16383。每个节点负责维护一部分槽以及槽所映射的 **键值数据**，如图所示：

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image004.jpg)

# 3 Redis集群搭建

## 3.1 环境准备

### 3.1.1 安装GCC

**Redis 安装需要依托GCC环境**

\1.   检查是否安装gcc

 命令: gcc -v

 如果能输出gcc版本信息,，说明安装了gcc。反之需要安装gcc

\2.   安装gcc

创建目录/usr/local/gccSrc

上传以下rpm文件至/usr/local/gccSrc目录下

cpp-4.8.5-11.el7.x86_64.rpm

gcc-4.8.5-11.el7.x86_64.rpm

glibc-devel-2.17-157.el7.x86_64.rpm

glibc-headers-2.17-157.el7.x86_64.rpm

kernel-headers-3.10.0-514.el7.x86_64.rpm

libmpc-1.0.1-3.el7.x86_64.rpm

mpfr-3.1.1-4.el7.x86_64.rpm

\3.   依次进行安装

rpm -ivh mpfr-3.1.1-4.el7.x86_64.rpm

rpm -ivh libmpc-1.0.1-3.el7.x86_64.rpm

rpm -ivh kernel-headers-3.10.0-514.el7.x86_64.rpm

rpm -ivh glibc-headers-2.17-157.el7.x86_64.rpm

rpm -ivh glibc-devel-2.17-157.el7.x86_64.rpm

rpm -ivh cpp-4.8.5-11.el7.x86_64.rpm

rpm -ivh gcc-4.8.5-11.el7.x86_64.rpm

\4.   都安装成功后，验证gcc -v

### 3.1.2 安装zlib

**因为Redis集群环境需要使用到ruby的接口，需要使用到redis的gem包，而安装gem包时需要有zlib的环境**

1、上传zlib-1.2.11.tar.gz到/usr/local目录下

2、解压 tar -xzvf zlib-1.2.11.tar.gz

3、cd zlib-1.2.11

4、./configure --prefix=/usr/local/zlib

（如果报错check for gcc，需要先安装gcc，安装完GCC再来执行）

5、make

6、make install

 

### 3.1.3 安装openssl

**因为Redis集群环境需要使用到ruby的接口，需要使用到redis的gem包，而安装gem包时需要有openssl的环境**

1、上传openssl-1.0.1h.tar.gz到/usr/local/目录下

2、解压 tar -xzvf openssl-1.0.1h.tar.gz

3、cd openssl-1.0.1h/

4、./config -fPIC --prefix=/usr/local/openssl enable-shared

5、./config -t

6、make && make install

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image006.jpg)

7、which openssl    //查看系统openssl的路劲

8、rm -rf /usr/bin/openssl   // /usr/bin/openssl是which openssl 查看到的旧版本的openssl的路径

9、cp /usr/local/openssl/bin/openssl /usr/bin/openssl //拷贝新版本的安装路径到系统路径

10、执行openssl version

输出版本信息

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image007.png)

### 3.1.4 安装ruby

**在安装Redis集群环境是需要使用到Ruby**

1、上传ruby-2.5.7.tar.gz到/usr/local目录下

2、解压 tar -xzvf ruby-2.5.7.tar.gz

3、cd ruby-2.5.7

4、./configure –-prefix=/usr/local/ruby  //-prefix是将ruby安装到指定目录

5、make && make install

成功安装如图所示

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image009.jpg)

6、配置环境变量

vi /etc/profile

添加环境变量，如图所示

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image011.jpg)

7、执行命令使配置文件生效

source /etc/profile

8、执行命令校验ruby是否安装成功，安装配置成功会输出ruby版本信息

ruby -v

 

9、进入/usr/local/ruby-2.5.7/ext/zlib/目录下

cd /usr/local/ruby-2.5.7/ext/zlib/ 

(备注：/usr/local/ruby-2.5.7这个目录是ruby安装包后解压的目录)

10、执行命令:ruby extconf.rb --with-zlib-include=/usr/local/zlib/include/ --with-zlib-lib=/usr/local/zlib/lib

（备注:/usr/local/zlib是zlib安装目录，命令执行后会生成Makefile文件）

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image013.jpg)

11、打开Makefile文件，找到下面一行(在文件倒数第3行)把路径修改一下

把zlib.o: $(top_srcdir)/include/ruby.h 改zlib.o: ../../include/ruby.h

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image014.png)

 

12、make && make install

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image016.jpg)

 

13、进入 /usr/local/ruby-2.5.7/ext/openssl/目录下

 cd /usr/local/ruby-2.5.7/ext/openssl/

14、执行命令:

ruby extconf.rb --with-openssl-include=/usr/local/openssl/include/ --with-openssl-lib=/usr/local/openssl/lib

（备注:/usr/local/openssl是openssl的安装目录，命令执行后会生成Makefile文件）

15、打开Makefile文件，在topdir = /usr/local/ruby/include/ruby-xxx后面新增一行 top_srcdir = /usr/local/ruby-2.5.7 

（备注/usr/local/ruby-2.5.7是ruby解压的目录）

 

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image018.jpg)

 

16、make && make install

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image020.jpg)

### 3.1.5 安装redis-4.1.3.gem

**安装集群环境时需要使用到ruby的接口，这个redis-4.1.3.gem提供了这个接口**

1、上传redis-4.1.3.gem到/usr/local/目录下

2、执行命令 gem install redis-4.1.3.gem

成功安装如图所示

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image022.jpg)

 

## 3.2 安装单机Redis

1、创建/usr/local/redis目录，上传redis-4.0.14.tar.gz至/usr/local/redis目录

2、cd /usr/local/redis

3、解压 tar -xzvf redis-4.0.14.tar.gz

4、cd redis-4.0.14/

5、make && make install

看到如下信息，redis安装成功了

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image024.jpg)

进入安装目录 cd /usr/local/redis/redis-4.0.14

6、修改配置文件redis.conf

bind 127.0.0.1 改为bind 0.0.0.0 【说明:表示所有ip都可以连接这个redis】

daemonize no 改为 daemonize yes 【说明：启动守护进程】

protected-mode yes 改为 protected-mode no 【说明：取消保护模式，如果启用保护模式需要设置密码】

logfile "" 改为 logfile "redis日志存储文件路径" 【说明：指定日志存储路径】

 

7、启动redis的命令

进入redis安装目录/usr/local/redis/redis-4.0.14，执行如下命令启动redis

redis-server redis.conf

启动后执行如下命令连接redis

redis-cli -p 6379 -h 192.169.1.86

(备注：6379是redis默认端口 192.169.1.86是redis所在服务器ip)

并执行命令set 1 1存入值至redis中测试一下

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image025.png)

 

## 3.3 部署集群环境

集群模式相当于将数据槽分片。每个节点分一段数据片。这样的话，当一个节点宕机后，这个节点没有备份的话，此段分片将不再可以使用。所以，官方推荐，集群内的每个节点都应该配备一个从节点，作为冷备。

官网推荐的模式，是三主三从的集群部署方式，这里以配置三主三从为例.

 

需要配置的节点信息

| **节点名称**  | **端口号** | **是主是从** | **所属主节点** |
| ------------- | ---------- | ------------ | -------------- |
| **Redis7001** | 7001       | 主           |                |
| **Redis7002** | 7002       | 主           |                |
| **Redis7003** | 7003       | 主           |                |
| **Redis7004** | 7004       | 从           | Redis7004      |
| **Redis7005** | 7005       | 从           | Redis7001      |
| **Redis7006** | 7006       | 从           | Redis7002      |

 

1、进入/usr/local/redis/目录

cd /usr/local/redis

2、执行mkdir redis700{1,2,3,4,5,6} 创建文件夹redis7001、redis7002、redis7003、redis7004、redis7005、redis7006

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image027.jpg)

3、将/usr/local/bin/ 目录下的redis-cli和redis-server复制到redis7001、...redis7006目录下

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image029.jpg)

 

1、在redis7001目录下创建redis.conf文件

vim redis.conf

文件内容

> \#端口号(6个对应各自的端口号)
>
> port 7001
>
> appendonly yes
>
> \#启动集群
>
> cluster-enabled yes
>
> \#yes 启用守护进程
>
> daemonize yes
>
> \#关联集群配置文件
>
> cluster-config-file "nodes.conf"
>
> \#设置超时
>
> cluster-node-timeout 5000
>
> \#日志信息
>
> logfile "redis7001.log"
>
> \#指定访问地址
>
> bind 0.0.0.0
>
> tcp-keepalive 300

 

redis7002目录下创建redis.conf文件

文件内容为

> port 7002
>
> appendonly yes
>
> cluster-enabled yes
>
> daemonize yes
>
> cluster-config-file "nodes.conf"
>
> cluster-node-timeout 5000
>
> logfile "redis7002.log"
>
> bind 0.0.0.0
>
> tcp-keepalive 300

 

依次类推，redis7003至redis7006目录下都创建redis.conf文件



每个目录下(redis7001至redis7006)都应该有这三个文件

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image030.png)

 

分别将这6个服务启动起来，启动命令:redis-server redis.conf

一个一个启动有点麻烦，在/usr/local/redis/目录下创建一下sh脚本来启动redis实例

vim startall.sh

> cd redis7001
>
> ./redis-server redis.conf
>
> cd ..
>
> cd redis7002
>
> ./redis-server redis.conf
>
> cd ..
>
> cd redis7003
>
> ./redis-server redis.conf
>
> cd ..
>
> cd redis7004
>
> ./redis-server redis.conf
>
> cd ..
>
> cd redis7005
>
> ./redis-server redis.conf
>
> cd ..
>
> cd redis7006
>
> ./redis-server redis.conf

 

赋予 startall.sh脚本可执行权限

chmod 777 startall.sh

执行 startall.sh脚本

./startall.sh

 

执行ps -ef|grep redis情况查看redis运行情况，启动成功如图所示

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image032.jpg)

 

创建集群，将这几个节点加入集群。首先进入redis-trib.rb所在目录

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image033.png)

进入/usr/local/redis/redis-4.0.14/src/目录

cd /usr/local/redis/redis-4.0.14/src/

执行如下命令:

./redis-trib.rb create --replicas 1 192.169.1.86:7001 192.169.1.86:7002 192.169.1.86:7003 192.169.1.86:7004 192.169.1.86:7005 192.169.1.86:7006

【备注:192.169.1.86是redis所在服务器ip,注意这个ip不能写127.0.0.1,否则只有本机才能连接】

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image035.jpg)

输入yes 回车 ，redis-trib.rb 开始执行 **节点握手** 和 **槽分配** 操作，如图所示加入集群成功

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image036.png)

说明:M表示是主节点，S表示是从节点】

执行 **集群检查**，检查各个 redis 节点占用的 **哈希槽**（slot）的个数以及 slot **覆盖率**。16384 个槽位中，**主节点Redis7001**、**主节点Redis7002** 和，**主节点Redis7003** 分别占用了 5461、5462 和 5461 个槽位。

 

连接集群测试

连接命令:redis-cli -p 其中一个节点的端口 -h 其中一个节点的ip -c

[备注：一定要有-c , -c表示以集群方式连接，没有-c就是单点连接了]

并往redis存一个值进行测试:set a a ,如下图表示成功

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image037.png)

## 3.3 集群环境中每个节点加入密码

加入密码有两种方式：

u 方法一：修改所有Redis集群中的redis.conf配置文件

```
masterauth 123456 //设置master密码，是为了Salve能够连接上Master

requirepass 123456 //设置Redis访问请求密码
```

u 方法二：进入各个Redis集群中的实时配置

```
./redis-cli -c -p 7001 //分别进入各个Redis片机进行各自设置

config set masterauth 123456 

config set requirepass 123456 

config rewrite 
```

注意：各个节点密码都必须一致，否则Redirected就会失败。推荐使用第一种方式。

先关闭所有的redis服务，使用命令 redis-cli -p 7001 shutdown依次将redis实例关闭。经过上面的步骤其实已经生成了相应的节点信息文件（nodes.conf）和数据信息文件（appendonly.aof，dump.rdb）。将所有redis7001 到redis7006的文件夹下的这三个文件都删除，然后修改redis.conf添加密码如图所示

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image039.jpg)

然后找到client.rb文件（可以使用 find / -name “client.rb”命令）然后修改password，如图

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image041.jpg)

然后使用脚本启动所有的集群节点。

再次运行命令 ./redis-trib.rb create --replicas 1 192.169.1.21:7001 192.169.1.21:7002 192.169.1.21:7003 192.169.1.21:7004 192.169.1.21:7005 192.169.1.21:7006 创建集群。

**测试集群**

使用命令 redis-cli -c -p 7001 设置一个set name song，会提示需要认证 输入auth “123456”即可 ，然后再次set name song 会redirected 到7002，也需要密码 ，当然可以直接再次使用 auth “123456”,这样显得有些繁琐，可以直接使用命令，redis-cli -c -p 7001 -a 123456连接（-a 相当于是输入密码验证），这样就可以直接跳转。如图

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image043.jpg) 

加入密码后的关闭redis实例的脚本也要相应的修改一下，加入-a 123456 的参数  修改后内容如下：

![img](D:\MyStudy\学习杂记\Redis\Redis集群环境部署.assets\clip_image045.jpg)