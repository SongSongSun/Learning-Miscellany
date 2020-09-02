## 1.1Set (集合)

**集合**（set）类型也是用来保存多个 **字符串元素**，但和 **列表类型** 不一样的是，集合中 **不允许有重复元素**，并且集合中的元素是 **无序的**，不能通过 **索引下标** 获取元素。

一个 **集合** 最多可以存储 2 ^ 32 - 1 个元素。Redis 除了支持 **集合内** 的 **增删改查**，同时还支持 **多个集合** 取 **交集**、**并集**、**差集**。合理地使用好集合类型，能在实际开发中解决很多实际问题。

### 1.1.1 相关命令

####    添加元素

命令为： **sadd key element [element ...]**

返回结果为添加成功的 **元素个数**，例如：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image002.jpg)

 

#### (1)  删除元素

命令为： **srem key element [element ...]**

返回结果为成功删除 **元素个数，**例如：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image003.png)

 

#### (2)  计算元素个数

命令为： **scard key**

scard 的 **时间复杂度** 为 O（1），它 **不会遍历** 集合所有元素，而是直接用 Redis 的 **内部** 的变量。

 

#### (3)  判断元素是否在集合中

命令为： sismember key element

如果给定元素 element 在集合内返回 1，反之返回 0

 

#### (4)  随机从集合返回指定个数元素

命令为： **srandmember key [count]**

[count] 是 **可选参数**，如果不写默认为 1

先将集合中的元素变成 a b c d e f 

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image005.jpg)

 

#### (5)  从集合随机弹出元素

命令为：**spop key**

spop 操作可以从 **集合** 中 **随机弹出** 一个元素

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image006.png)

**注意**：Redis 从 3.2 版本开始， spop 也支持 [count] 参数。

srandmember 和 spop 都是 **随机** 从集合选出元素，两者不同的是 spop 命令执行后，**元素** 会从集合中 **删除**，而 srandmember 不会删除元素

 

#### (6)  获取所有元素

命令为： **smembers key**

返回集合的**所有元素**，并且返回结果是**无序**的

 

#### (7)  求多个集合的交集

现在有 两个集合，它们分别是 user:1:follow 和 user:2:follow。

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image008.jpg)

 

命令为： **sinter key [key ...]**

使用效果如图：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image010.jpg)

 

#### (8)  求多个集合的并集

命令为： **suinon key [key ...]**

使用效果如图

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image012.jpg)

 

#### (9)  求多个集合的差集

命令为：**sdiff key [key ...]**

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image014.jpg)

前面三个求 **交集**、**并集** 和 **差集** 的操作得到的结果，如图所示
 ![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image016.jpg)

#### (10) 将交集，并集，差集结果保存

**集合间** 的运算在 **元素较多** 的情况下会 **比较耗时**，所以 Redis 提供了以下 **三个命令**（**原命令** + store）将 **集合间交集**、**并集**、**差集** 的结果保存在 destination key 中。

命令为： 

**sinterstore destination key [key ...]** 

**suionstore destination key [key ...]**

**sdiffstore destination key [key ...]**

 

下面的操作会将 user:1:follow 和 user:2:follow 两个集合 的 交集结果 保存在 user:1_2:inter 中，user:1_2:inter 本身也是 **集合类型**

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image018.jpg)

#### (11) 命令的时间复杂度

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Set.assets\clip_image020.jpg)

 

### 1.1.2 应用场景

####  (1)用户标签

举两个例子：

**娱乐新闻推荐：**一个用户可能对娱乐、体育比较感兴趣，另一个用户可能对历史、新闻比较感兴趣，这些兴趣点就是标签。有了这些数据就可以得到喜欢同一个标签的人，以及用户的共同喜好的标签，这些数据对于用户体验以及增强用户黏度比较重要。

**电商人群分类：**一个电子商务的网站会对不同标签的用户做不同类型的推荐，比如对数码产品比较感兴趣的人，在各个页面或者通过邮件的形式给他们推荐最新的数码产品，通常会为网站带来更多的利益。

 

#### (2) 其他应用场景

| **命令组合**             | **应用场景**         |
| ------------------------ | -------------------- |
| **sadd**                 | 标签                 |
| **spop /   srandmember** | 生成随机数（如抽奖） |
| **sadd + sinter**        | 社交需求（共同好友） |