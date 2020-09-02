## 1 String (字符串类型)

**字符串类型** 是 Redis 最基础的数据结构。**字符串类型** 的值实际可以是 **字符串**（**简单** 和 **复杂** 的字符串，例如 JSON、XML）、**数字**（整数、浮点数），甚至是 **二进制**（图片、音频、视频），但是值最大不能超过 512MB

### 1.1相关命令

#### (1)  设置值

命令为：**set key value [ex seconds] [px milliseconds] [nx|xx]**

set 命令有几个选项：

l **ex seconds**：为键设置秒级过期时间。

l **px milliseconds**：为键设置毫秒级过期时间。

l **nx**：键必须**不存在**，才可以设置成功，用于添加。

l **xx**：与 nx 相反，键必须**存在**才可以设置成功，用于更新。

使用方法如图：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image002.jpg)

 

除了 set 选项，Redis 还提供了 setex 和 setnx 两个命令：

**setex key seconds value**

**setnx key value**

 

**setex：**设定键的值，并指定此键值对应的 有效时间**。**

使用方法如图：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image004.jpg)

 

**setnx：**键必须 不存在，才可以设置成功。如果键已经存在，返回 0**。**

使用方法如图：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image006.jpg)

 

#### (2)  获取值

命令为：**get key**

如果要获取的 **键不存在**，则返回 nil（**空**）。

 

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image007.png)

 

#### (3)  批量设置值

命令为：**mset key value [key value ...]**

 

下面操作通过 mset 命令一次性设置 4 个 键值对：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image008.png)

 

#### (4)  批量获取值

命令为：**mget key [key ...]**

 

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image010.jpg)

 

**批量操作**命令，可以有效提高**开发效率**，假如没有 mget 这样的命令，要执行 n 次 get 命令的过程和耗时如下：

**n次get时间 = n次网络时间 + n次命令时间**

 

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image012.jpg)

 

使用 mget 命令后，执行 n 次 get 命令的过程和 **耗时** 如下：

**n次get时间 = 1次网络时间 + n次命令时间**

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image014.jpg)

 

#### (5)  计数

命令为： **incr key**

incr 命令用于对值做**自增**操作，返回结果分为三种情况：

l 值不是 整数，返回 错误。

l 值是 整数，返回 自增 后的结果。

l 键不存在，按照值为 0自增，返回结果为 1。

使用如图

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image016.jpg)

 

除了 incr 命令，Redis 还提供了 decr（**自减**）、incrby（**自增指定数字**）、decrby（**自减指定数字**）、incrbyfloat（**自增浮点数**）等命令操作。

 

#### (6)  各种命令的时间复杂度

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image018.jpg)

 

### 1.2 应用场景

####    缓存功能

下面是一种比较典型的缓存使用场景，其中Redis作为**缓存层**，MySQL 作为存储层，绝大部分请求的数据都是从Redis中获取。由于Redis具有支撑高并发的特性，所以缓存通常能起到**加速读写**和**降低后端压力**的作用。

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image020.jpg)

 

#### (1)  计数

许多应用都会使用 Redis 作为 计数 的基础工具，它可以实现 快速计数、查询缓存 的功能，同时数据可以 异步落地 到其他 数据源。一般来说，视频播放数系统，就是使用 Redis 作为 视频播放数计数 的基础组件，用户每播放一次视频，相应的视频播放数就会自增 1。

 

一个真实的 **计数系统** 要考虑的问题会很多：**防作弊**、按照 **不同维度** 计数，**数据持久化** 到 **底层数据源**等。

 

#### (2)  共享Session

一个 分布式 Web 服务将用户的 Session 信息（例如 用户登录信息）保存在 各自 的服务器中。这样会造成一个问题，出于 负载均衡 的考虑，分布式服务 会将用户的访问 均衡 到不同服务器上，用户 刷新一次访问 可能会发现需要 重新登录，这个问题是用户无法容忍的。

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image022.jpg)

为了解决这个问题，可以使用 Redis 将用户的 Session 进行 集中管理。在这种模式下，只要保证 Redis 是 高可用 和 扩展性的，每次用户 更新 或者 查询 登录信息都直接从 Redis 中集中获取。
 ![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之String.assets\clip_image024.jpg)

#### (3)  限速

很多应用出于安全的考虑，会在每次进行登录时，让用户输入手机验证码，从而确定是否是用户本人。但是为了短信接口不被频繁访问，会限制用户每分钟获取验证码的频率。例如一分钟不能超过 5 次。就可以通过设置redis键的过期时间搭配incr指令来完成限速。



参考

[深入剖析Redis系列]: https://zhuanlan.zhihu.com/p/45699203

