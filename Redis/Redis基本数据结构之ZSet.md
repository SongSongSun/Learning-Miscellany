## 1.1Zset(有序集合)

Zset保留了集合不能有重复成员的特性，但不同的是，有序集合中的元素可以排序。但是它和列表使用索引下标作为排序依据不同的是，它给每个元素设置一个分数（score）作为排序的依据。

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image002.jpg)

有序集合中的元素不能重复，但是score可以重复，就和一个班里的同学学号不能重复，但是考试成绩可以相同。

 

### 1.1.1 相关指令

####    添加元素

命令为：**zadd key score member [score member ...]**

下面操作向有序集合user:ranking添加用户tom和他的分数251：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image004.jpg)

返回结果代表**成功添加成员的个数**

 

有关zadd命令有两点需要注意：

l Redis3.2为zadd命令添加了nx、xx、ch、incr四个选项：

nx：member必须不存在，才可以设置成功，用于添加。

xx：member必须存在，才可以设置成功，用于更新。

ch：返回此次操作后，有序集合元素和分数发生变化的个数

incr：对score做增加，相当于后面介绍的zincrby。

 

l 有序集合相比集合提供了排序字段，但是也产生了代价，zadd的时间

复杂度为**O（log（n））**，sadd的时间复杂度为**O（1）**。

 

#### (1)  计算成员个数

命令为：zcard key

**zcard的时间复杂度为O（1）**

 

#### (2)  计算某个成员的分数

命令为：**zscore key member**

使用效果如图

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image006.jpg)

成员不存在则返回nil

 

#### (3)  计算成员的排名

命令为：

**zrank key member**

**zrevrank key member**

 

zrank是从分数**从低到高**返回排名，zrevrank反之。

 

#### (4)  删除成员

命令为：**zrem key member [member ...]**

返回的结果为成功删除的个数

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image007.png)

 

#### (5)  增加成员的分数

命令为：**zincrby key increment member**

使用效果：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image009.jpg)

 

#### (6)  返回指定排名范围的成员

命令为： **zrange key start end [withscores]**  **zrevrange key start end [withscores]**

有序集合是按照分值排名的，zrange是从低到高返回，zrevrange反之。下面代码返回排名最低的是三个成员，如果加上withscores选项，同时会返回成员的分数：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image011.jpg)

 

#### (7)  返回指定分数范围的成员

命令为：

**zrangebyscore key min max [withscores] [limit offset count]**

**zrevrangebyscore key max min [withscores] [limit offset count]**

 

其中zrangebyscore按照分数从低到高返回，zrevrangebyscore反之。例如下面操作从低到高返回200到221分的成员，withscores选项会同时返回每个成员的分数。**[limit offset count]**选项可以**限制输出的起始位置和个数**。同时min和max还支持开区间（小括号）和闭区间（中括号），-inf和+inf分别代表无限小和无限大

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image013.jpg)

 

#### (8)  删除指定排名内的升序元素

命令为： **zremrangebyrank key start end**

下面操作删除第start到第end名的成员：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image015.jpg)

 

#### (9)  删除指定分数范围的成员

命令为：**zremrangebyscore key min max**

使用效果如图：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image017.jpg)

 

#### (10) 交集

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image019.jpg)

将如图所示的zset导入redis

 

交集命令为：**zinterstore destination numkeys key [key ...] [weights weight [weight ...]]**

**[aggregate sum|min|max]**

 

这个命令参数较多，下面分别进行说明：

l **destination**：交集计算结果保存到这个键。

l **numkeys**：需要做交集计算键的个数。

l **key[key...]**：需要做交集计算的键。

l **weights weight[weight...]**：每个键的权重，在做交集计算时，每个键中的每个member会将自己分数乘以这个权重，每个键的权重默认是1。

l **aggregate sum|min|max**：计算成员交集后，分值可以按照sum（和）、min（最小值）、max（最大值）做汇总，默认值是sum。

 

下面操作对user：ranking：1和user：ranking：2做交集，weights和aggregate使用了默认配置，可以看到目标键user：ranking：1_inter_2对分值做了sum操作

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image021.jpg)

 

#### (11) 并集

命令为： **zunionstore destination numkeys key [key ...] [weights weight [weight ...]] [aggregate sum|min|max]**

 

该命令的所有参数和zinterstore是一致的，只不过是做并集计算，例如下面操作是计算user：ranking：1和user：ranking：2的并集，weights和aggregate使用了默认配置，可以看到目标键user：ranking：1_union_2对分值做了sum操作

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image023.jpg)

 

#### (12) 命令时间复杂度

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之ZSet.assets\clip_image025.jpg)

 

### 1.1.2 应用场景（排行榜）

有序集合比较典型的使用场景就是排行榜系统。例如视频网站需要对用户上传的视频做排行榜，榜单的维度可能是多个方面的：按照时间、按照播放数量、按照获得的赞数。