# MySQL的逻辑架构
## Server层
### 连接器

负责跟客户端建立连接、获取权限、维持和管理连接。
**mysql -h主机地址 -u用户名 -p用户密码**
p后可直接写密码，但可能造成密码泄露
```sql
eg:
主机的IP为：10.0.0.1，用户名为root,密码为123：
mysql -h10.0.0.1 -uroot -p
Enter password:123
```

一个用户成功建立连接后，即使管理员账号对这个用户的权限做了修改，也不会影响已经存
在连接的权限。修改完成后，只有再新建的连接才会使用新的权限设置。

#### 连接
##### 长连接

连接成功后如果客户端持续有请求则一直使用同一个连接

##### 短链接

每次执行完很少几次查询就断开连接，下次查询**重新建立连接**


连接通常比较复杂繁琐，应当尽量使用**长连接**，但如果全部使用长连接则MySQL占用内存涨的很快，因为MySQL在执行过程中临时使用的内存管理在连接对象中，资源在断开连接时才释放，如果长连接累积, 导致内存占用太大，被系统强行杀掉(OOM)，看上去就是MySQL**异常重启。**

##### 解决方法
**1.** 定期断开长连接，使用一段时间，或者程序里面判断执行过一个占用内存的查询后，断开连接，之后等要查询时再重新连接。
**2.** MySQL5.7以上版本可以每次执行一个较大的操作后，通过执行mysql_reset_connection来重新初始化连接资源，此过程不需要重连和重新坐权限验证，但是会将连接恢复到刚刚创建完时的状态。

### 查询缓存

MySQL在拿到查询时，会先看看查询缓存，看看之前是否执行过这条语句(之前执行过的语句可能会以Key-Value的形式直接被缓存在内存中，**如果结果在缓存中**则value会直接返回到客户端；**如果结果不在缓存中**则执行后面的阶段(分析器、优化器、执行器)）。

虽然使用查询缓存后查询效率很高，但是不建议使用，因为**失败概率很大**。只要对表上的一个内容更新，**整张表的查询缓存**都被清空。除非是很长时间才更新一次的静态表。

注：MySQL8.0之后的查询缓存已经被删除了。

### 分析器

先做词法分析后做语法分析

### 优化器
在表里有多个索引时判断使用哪一个索引，或在一个语句有多个关联(join)时确定各个表的**连接顺序**(让其正确、高效)。

### 执行器
分析器告诉你做什么，优化器告诉你怎么做，执行器则是执行语句。执行时会判断你对这个表是否有执行相关语句的权限，如果没有则报错返回，如果有则执行。

## 存储引擎层

## MySQL语法
```sql
[]用于指明可选字或子句
当某一语法成分由多个可选项组成时，可选项应用竖线“|”分开
当必须选择一组选择中的某一成员时，可选项将列在{}中
    eg: {a|b}...... a或b其中一个必须被选择
用;来区分每一条语句
-- 注释 
```
### 数据库的创建
```sql
CREATE DATABASE databaseName;
```
```sql
mysqladmin -u root -p create databaseName
Enter password:******
```
### 数据库的删除
```sql
DROP DATABASE databaseName;
```
```sql
mysqladmin -u root -p drop databaseName
Enter password:******
```

### 数据类型

#### 数值类型
```sql
类型                大小(Byte)        
TINYINT             1      
SMALLINT            2      
INT 或 INTEGER      4       
BIGINT              8     
FLOAT               4
DOUBLE              8
DECIMAL     DECIMAL (a, b)  如果a>b, 为a+2否则为b+2
```
#### 时间类型
```sql
类型        大小(Byte)  格式
DATE        3           YYYY-MM-DD
TIME        3           HH:MM:SS
YEAR        1           YYYY
DATETIME    8           YYYY-MM-DD hh:mm:ss '1000-01-01 00:00:00' 到 '9999-12-31 23:59:59'	
TIMESTAMP   4           YYYY-MM-DD hh:mm:ss '1970-01-01 00:00:01' UTC 到 '2038-01-19 03:14:07' UTC
```

#### 字符串类型
```sql
用单引号或双引号括起来的字符序列
一个字符串文字可能有一个可选的字符集引介词和COLLATE子句
引介词：告诉解析程序，“后面将要出现的字符串使用字符集X。”
COLLATE：排序规则，告知mysql如何进行排序和比较
```
```sql
类型        最大大小(Byte)  
CHAR        255         定长字符串
VARCHAR     65535       变长字符串
TINYBLOB    255         二进制字符串
TINYTEXT    255         文本字符串
BLOB        65535
TEXT        65535
MEDIUMBLOB  2^24        
MEDIUMTEXT  2^24
LONGBLOB    2^32
LONGTEXT    2^32
```

### 创建

#### 创建数据表
```sql
CREATE TABLE tableName ( columnName columnType, ......);
```

### 优化
#### WHERE优化
**1.** 去除不必要的括号
```sql
((a AND b) AND c OR (((a AND b) AND (c AND d))))
改为
(a AND b AND c) OR (a AND b AND c AND d)
```
**2.** 常量重叠
```sql
(a<b AND b=c) AND a=5
改为
b>5 AND b=c AND a=5
```

**3.** 去除常量条件
```sql
(B>=5 AND B=5) or (B=6 AND 5=5) OR (B=7 AND 5=6)
改为
B=5 OR B=6
```
#### 范围优化
**1.** 用常量替换条件
**2.** 去除总是为``TRUE``或``FALSE``的条件
**3.** 删不必要的``TRUE``和``FALSE``常量

#### 索引合并

### 查询
```sql
SELECT * 查询整个列表
SELECT xxx  查询xxx，xxx也可以使用加减乘除取模
DISTINCT 不显示重复项
xxx AS yyy  yyy为xxx的描述性名称 yyy可为字符串
FROM 指明要查询的表
WHERE 查找条件，用于筛选数据
ORDER BY 排序方式
REGEXP xxx 查询使用xxx正则表达式
```