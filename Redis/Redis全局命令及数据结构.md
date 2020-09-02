# 1. Redis全局命令

Redis有5中数据结构，他们是键值对中的值，但是对于键来说有一些通用的命令

## 1.1 查看所有的键

命令为：**keys \***

 

使用方法如图：

![img](D:\MyStudy\学习杂记\Redis\Redis全局命令及数据结构.assets\clip_image002.jpg)

 

## 1.2 键总数

命令为： **dbsize**

 

使用方法如图：

![img](D:\MyStudy\学习杂记\Redis\Redis全局命令及数据结构.assets\clip_image004.jpg)

dbsize 命令在 **计算键总数** 时 **不会遍历** 所有键，而是直接获取 Redis **内置的键总数变量**，所以 dbsize 命令的 时间复杂度 是 O（1）。而 keys 命令会**遍历所有键**，所以它的时间复杂度 是 O（n），当 Redis 保存了大量键 时，线上环境禁止使用

 

## 1.3 检查键是否存在

命令为： **exists key**

如果键存在返回1，不存在返回0

 

使用方法如图：

![img](D:\MyStudy\学习杂记\Redis\Redis全局命令及数据结构.assets\clip_image006.jpg)

 

## 1.4 删除键

命令为: **del key**

del 是一个通用的命令，无论是值是什么数据结构类型，del命令都可以将它删除

 

使用方法如图：

![img](D:\MyStudy\学习杂记\Redis\Redis全局命令及数据结构.assets\clip_image008.jpg)

返回结果为 成功删除 的 键的个数，假设删除一个 不存在 的键，就会返回 0

 

## 1.5 键过期

命令为：**expire key seconds**

Redis 支持对键添加 过期时间，当超过过期时间后，会**自动删除键**。例如为键 hello 设置 10 秒过期时间：

![img](D:\MyStudy\学习杂记\Redis\Redis全局命令及数据结构.assets\clip_image010.jpg)

 

ttl 命令会返回键的 **剩余过期时间**，它有 3 种返回值：

- 大于等于 0 的整数：表示键 剩余 的 过期时间。
- 返回 -1：键 没设置 过期时间。
- 返回 -2：键 不存在。

 

## 1.6 键的数据结构类型

命令为：**type key**

会返回键对应的**值的数据结构类型** 键不存在返回none

 

使用方法如图:

![img](D:\MyStudy\学习杂记\Redis\Redis全局命令及数据结构.assets\clip_image012.jpg)

# 2. Redis数据结构

Redis有5中基本的数据结构，分别是string(字符串),hash(哈希),list(列表),set(集合),zset(有序集合)。如图所示：

![img](D:\MyStudy\学习杂记\Redis\Redis全局命令及数据结构.assets\clip_image014.png)

下一篇将详细介绍各类数据结构