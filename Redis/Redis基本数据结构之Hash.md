## 1Hash (哈希)

在 Redis 中，**哈希类型** 是指键值本身又是一个 **键值对结构**。**哈希** 形如 value={ {field1，value1}，...{fieldN，valueN} }，Redis **键值对** 和 **哈希类型** 二者的关系如图所示：

 

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image002.jpg)

 

哈希类型中的 **映射关系** 叫作 field-value，这里的 value 是指 field 对应的 **值**，不是 **键** 对应的值。

### 1.1 相关命令

####    设置值

命令为： **hset key field value**

使用方法

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image004.jpg)

此外 Redis 提供了 hsetnx 命令，它们的关系就像 set 和 setnx 命令一样，只不过 作用域 由 键 变为 field。

#### (1)  获取值

命令为：**hget key field**

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image006.jpg)

#### (2)  删除field

命令为： **hdel key field [field ...]**

hdel 会删除 **一个或多个** field，返回结果为 **成功删除** field 的个数

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image007.png)

 

#### (3)  计算field个数

命令为： **hlen key**

 

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image009.jpg)

 

#### (4)  批量设置或获取field-value

批量设置命令为：**hmset key field value [field value ...]**

批量获取命令为：**hmget key field [field ...]**

 

hmset 和 hmget 分别是 **批量设置** 和 **获取** field-value，hmset 需要的参数是 key 和 **多对** field-value，hmget 需要的参数是 key 和 **多个** field

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image011.jpg)

#### (5)  获取所有的field

命令为： **hkeys key**

返回指定 **哈希键** 所有的 field

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image012.png)

#### (6)  获取所有value

命令为：**hvals key**

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image013.png)

#### (7)  获取所有的field-value

命令为： **hgetall key**

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image014.png)

 

#### (8)  判断field是否存在

命令为： **hexists key field**

**key包含field 返回 1 不包含返回0**

 

#### (9)  各种命令的时间复杂度

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image016.jpg)

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image018.jpg)

 

### 1.2 应用场景

如图所示，为 关系型数据表 的两条 用户信息，用户的属性作为表的列，每条用户信息作为行。

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image020.jpg)

使用 Redis 哈希结构 存储 用户信息 的示意图如下：

![img](D:\MyStudy\学习杂记\Redis\Redis基本数据结构之Hash.assets\clip_image022.jpg)

 

相比于使用 字符串序列化 缓存 用户信息，**哈希类型 变得更加 直观，并且在 更新操作 上会 更加便捷**。可以将每个用户的 id 定义为 键后缀，多对 field-value 对应每个用户的 属性