##  List (列表)

**列表**（list）类型是用来存储多个 **有序** 的 **字符串**。在 Redis 中，可以对列表的 **两端** 进行 **插入**（push）和 **弹出**（pop）操作，还可以获取 **指定范围** 的 **元素列表**、获取 **指定索引下标** 的 **元素** 等。

**列表** 是一种比较 **灵活** 的 **数据结构**，它可以充当 **栈** 和 **队列** 的角色，在实际开发上有很多应用场景。

如图所示，a、b、c、d、e 五个元素 **从左到右** 组成了一个 **有序的列表**，列表中的每个字符串称为 **元素**（element），一个列表最多可以存储 2 ^ 32 - 1 个元素。

- 列表的 **插入** 和 **弹出** 操作

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image002.jpg)

- 列表的 **获取**、**截取** 和 **删除** 操作

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image004.jpg)

### 1相关命令

####    从右边插入元素

命令为： **rpush key value [value ...]**

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image006.jpg)

 

#### (1)  从左边插入元素

命令为： **lpush key value [value ...]**

使用方法和 rpush 相同，只不过从 **左侧插入**

 

#### (2)  向某个元素前或者后插入

命令为： **linsert key before|after pivot value**

 

linsert 命令会从 列表 中找到 **第一个 等于 pivot 的元素**，在其 前（before）或者 后（after）插入一个新的元素 value，例如下面操作会在列表的 元素 b 前插入 redis：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image008.jpg)

返回的结果代表**列表的长度**。

 

#### (3)  获取列表指定索引下标的元素

命令为： **lindex key index**

 

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image010.jpg)

 

#### (4)  获取列表长度

命令为： **llen key**

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image011.png) 

 

#### (5)  从左侧或右侧弹出元素

从左侧弹出命令为： **lpop key**

从右侧弹出命令为： **rpop key**

 

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image013.jpg)

#### (6)  删除指定元素

命令为： **lrem key count value**

lrem 命令会从 **列表** 中找到 **等于** value 的元素进行 **删除**，根据 count 的不同分为三种情况：

- **count > 0**：**从左到右**，删除最多 count 个元素。
- **count < 0**：**从右到左**，删除最多 count**绝对值** 个元素。
- **count = 0**，**删除所有**。

例如向列表 **从左向右** 插入 5 个 a，那么当前 **列表** 变为 “a a a a a redis b ”，下面操作将从列表 **左边** 开始删除 4 个为 a 的元素：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image015.jpg)

 

#### (7)  按照索引列表修剪列表

命令为: **ltrim key start stop**

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image017.jpg)

 

#### (8)  阻塞弹出命令

向左阻塞弹出命令： **blpop key [key ...] timeout**

向右阻塞弹出命令： **brpop key [key ...] timeout**

 

blpop 和 brpop 是 lpop 和 rpop 的 **阻塞版本**，它们除了 **弹出方向** 不同，**使用方法** 基本相同，所以下面以 brpop 命令进行说明， brpop 命令包含两个参数：

- **key[key...]**：一个列表的 **多个键**。
- **timeout**：**阻塞** 时间（单位：**秒**）。

对于 timeout 参数，要氛围 **列表为空** 和 **不为空** 两种情况：

l **列表为空**

如果 timeout = 3，那么 **客户端** 要等到 3 秒后返回，如果 timeout = 0，那么 **客户端** 一直 **阻塞** 等下去，如果在此期间添加了数据，客户端会立即返回

l **列表不为空**

客户端会 立即返回。

 

#### (9)  命令时间复杂度

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image019.jpg)

 

### 2 应用场景

####    消息队列

通过 Redis 的 lpush + brpop 命令组合，即可实现**阻塞队列**。如图所示：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之List.assets\clip_image021.jpg)

生产者客户端 使用 lrpush 从列表 左侧插入元素，多个消费者客户端使用 brpop 命令阻塞式的“抢”列表 尾部 的元素，多个客户端保证了消费的**负载均衡**和**高可用性**。

#### (10) 其他场景

| **命令组合**       | **对应数据结构** |
| ------------------ | ---------------- |
| **lpush +   lpop** | Stack（栈）      |
| **Lpush +   rpop** | Queue(队列)      |