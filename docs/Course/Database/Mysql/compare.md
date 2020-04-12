# Mysql的比较问题

> 本文作者 [@zww](http://www.wenweizeng.com/)

Mysql版本：8.0.12

## 字符串与数字的比较

> Comparison operations result in a value of 1 (TRUE), 0 (FALSE), or NULL. These operations work for both numbers and strings. Strings are automatically converted to numbers and numbers to strings as necessary. 
https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html#operator_greater-than

也就是说，在字符串与数字进行比较时，mysql会将字符串转化为数字进行比较

### 只含数字

``` sql
mysql> select '5' > 2;
+---------+
| '5' > 2 |
+---------+
|       1 |
+---------+
1 row in set (0.00 sec)

mysql> select '2' =  2;
+----------+
| '2' =  2 |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)

mysql> select '1123123' > 2;
+---------------+
| '1123123' > 2 |
+---------------+
|             1 |
+---------------+
1 row in set (0.00 sec)

mysql> select '000000000123123' > 2;
+-----------------------+
| '000000000123123' > 2 |
+-----------------------+
|                     1 |
+-----------------------+
1 row in set (0.00 sec)
```
普通的按照数值进行比较

### 含字母
``` sql
mysql> select 'aaa' = 2; <<
+-----------+
| 'aaa' = 2 |
+-----------+
|         0 |
+-----------+
1 row in set, 1 warning (0.00 sec)

mysql> select 'aaa' = 0;
+-----------+
| 'aaa' = 0 |
+-----------+
|         1 |
+-----------+
1 row in set, 1 warning (0.00 sec)

mysql> select '2aaa' = 2; <<
+------------+
| '2aaa' = 2 |
+------------+
|          1 |
+------------+
1 row in set, 1 warning (0.00 sec)

mysql> select 'aaa2' = 2;
+------------+
| 'aaa2' = 2 |
+------------+
|          0 |
+------------+
1 row in set, 1 warning (0.00 sec)

mysql> select 'aaa2' = 0;
+------------+
| 'aaa2' = 0 |
+------------+
|          1 |
+------------+
1 row in set, 1 warning (0.00 sec)

mysql> select '0002aaa' = 2;
+---------------+
| '0002aaa' = 2 |
+---------------+
|             1 |
+---------------+
1 row in set, 1 warning (0.00 sec)

mysql> select '231aaa' > 230; <<
+----------------+
| '231aaa' > 230 |
+----------------+
|              1 |
+----------------+
1 row in set, 1 warning (0.00 sec)

mysql> select '231a1111aa' > 232;
+--------------------+
| '231a1111aa' > 232 |
+--------------------+
|                  0 |
+--------------------+
1 row in set, 1 warning (0.00 sec)
```
从左到右转换为数值进行比较，直到字符便终止
如果转换不了，即为0

### 负数

``` sql

mysql> select '-1' = 0;
+----------+
| '-1' = 0 |
+----------+
|        0 |
+----------+
1 row in set (0.00 sec)

mysql> select '-1' = -1;
+-----------+
| '-1' = -1 |
+-----------+
|         1 |
+-----------+
1 row in set (0.00 sec)

mysql> select '-100' > -999;
+---------------+
| '-100' > -999 |
+---------------+
|             1 |
+---------------+
1 row in set (0.00 sec)

mysql> select '-900' > 1;
+------------+
| '-900' > 1 |
+------------+
|          0 |
+------------+
1 row in set (0.00 sec)

mysql> select '---9' = 0; << 不合法
+------------+
| '---9' = 0 |
+------------+
|          1 |
+------------+
1 row in set, 1 warning (0.00 sec)
```
如果是一个合法的负数字符串，还是可以正常比较的
如果不合法，则为0

### 精度丢失问题

``` sql
mysql> select '1111111111111111' = 1111111111111112; << 16位
+---------------------------------------+
| '1111111111111111' = 1111111111111112 |
+---------------------------------------+
|                                     0 |
+---------------------------------------+
1 row in set (0.00 sec)

mysql> select '11111111111111111' = 11111111111111112; << 17位
+-----------------------------------------+
| '11111111111111111' = 11111111111111112 |
+-----------------------------------------+
|                                       1 |
+-----------------------------------------+
1 row in set (0.00 sec)
```
mysql中字符串默认转化前16位，16位以后的部分就被截断了

## 字符串与字符串的比较

``` sql
mysql> select '05' > 4;
+----------+
| '05' > 4 |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)

mysql> select '05' > '4'; << ascii(0) < ascii(4)
+------------+
| '05' > '4' |
+------------+
|          0 |
+------------+
1 row in set (0.00 sec)

mysql> select 'a05' > '4'; << ascii(a) > ascii(4)
+-------------+
| 'a05' > '4' |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)

mysql> select '05a' > '4';
+-------------+
| '05a' > '4' |
+-------------+
|           0 |
+-------------+
1 row in set (0.00 sec)

mysql> select '-1' < '-2';
+-------------+
| '-1' < '-2' |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)
```
按照字符串每一位的ascii码值依次进行比较

### 大小写比较

``` sql
mysql> select 'G' > 'a';
+-----------+
| 'G' > 'a' |
+-----------+
|         1 |
+-----------+
1 row in set (0.00 sec)
```
明明ascii(G)是小于ascii(a)，为什么会返回1呢

这是因为 **mysql默认是不区分大小写的** ，所以说实际上比较的是ascii(g)与ascii(a)，当然前者大，返回1

如果我们要比较ascii码的大小，那我们就需要使用 **二进制字符串** 进行比较

具体来说，mysql可以使用三类函数进行类型的转换

| Name | Description |
| ------ | ------ |
| BINARY | Cast a string to a binary string |
| CAST() | Cast a value as a certain type |
| CONVERT() | Cast a value as a certain type |

所以我们只需要使用函数，转换字符串的类型即可实现比较 

``` sql
# 使用binary
mysql> select (binary 'G') < 'a';
+--------------------+
| (binary 'G') < 'a' |
+--------------------+
|                  1 |
+--------------------+
1 row in set (0.00 sec)

# 使用cast
mysql> select cast('G' as binary) < 'a';
+---------------------------+
| cast('G' as binary) < 'a' |
+---------------------------+
|                         1 |
+---------------------------+
1 row in set (0.00 sec)

# 使用convert
mysql> select convert('G', binary) < 'a';
+----------------------------+
| convert('G', binary) < 'a' |
+----------------------------+
|                          1 |
+----------------------------+
1 row in set (0.00 sec)

# 或者可以使用mysql8中新增的数据类型json，因为json的string类型是大小写敏感的

mysql> select concat('G', cast('0' as json)) < 'a';
+--------------------------------------+
| concat('G', cast('0' as json)) < 'a' |
+--------------------------------------+
|                                    1 |
+--------------------------------------+
1 row in set (0.00 sec)
```

或者在创建表的时候就指定collation

>mysql collation的命名规则：
以相关的字符集名开始，通常包括一个语言名，并且以_ci（大小写不敏感）、_cs（大小写敏感）或_bin（二元）结束
比如：COLLATE=utf8mb4_bin

## 数字与数字的比较

``` sql
mysql> select -1 < 2;
+--------+
| -1 < 2 |
+--------+
|      1 |
+--------+
1 row in set (0.00 sec)
```
支持负数的比较

### 定点数和浮点数

#### 定点数
在MySQL中，DECIMAL和NUMERIC是定点类型数据，以二进制形式存储，因此存储的是准确的数字
在创建MySQL的DECIMAL列的时候，可以指定进度和标度：DECIMAL(M,D)
M为精度(precision)，表示该值的总长度；D为标度(scale)，表示小数点后面的长度
例如：
DECIMAL(7,4)可存储的数据范围为-999.9999~999.9999。MySQL保存值时进行四舍五入，如果插入999.00009，则结果为999.0001。
在标准SQL中，语法DECIMAL为默认值，等价于DECIMAL(10,0)。DECIMAL(M)等价于DECIMAL(M,0)

``` sql
mysql> desc test;
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| value | decimal(5,2) | YES  |     | NULL    |       |
+-------+--------------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql> insert into test values (1.234);
Query OK, 1 row affected, 1 warning (0.06 sec)
# 小数点后最多2位，所以自动四舍五入数据截断，但会有warning

mysql> show warnings;
+-------+------+--------------------------------------------+
| Level | Code | Message                                    |
+-------+------+--------------------------------------------+
| Note  | 1265 | Data truncated for column 'value' at row 1 |
+-------+------+--------------------------------------------+
1 row in set (0.00 sec)

mysql> insert into test values (12.34);
Query OK, 1 row affected (0.09 sec)
# 没问题

mysql> insert into test values (1.2);
Query OK, 1 row affected (0.07 sec)
# 未满部分补0

mysql> insert into test values (1234.5);
ERROR 1264 (22003): Out of range value for column 'value' at row 1
# 小数部分未满2位，补0，所以保存应该1234.50
# 但因为1234.50的位数超出了5，报错

mysql> select * from test;
+-------+
| value |
+-------+
|  1.23 |
| 12.34 |
|  1.20 |
+-------+
3 rows in set (0.00 sec)
```

#### 浮点数
浮点数相对于定点数的优点是在长度一定的情况下，浮点数能够表示更大的数据范围，但是会引起精度问题

``` sql
mysql> desc test2;
+-------+-------+------+-----+---------+-------+
| Field | Type  | Null | Key | Default | Extra |
+-------+-------+------+-----+---------+-------+
| value | float | YES  |     | NULL    |       |
+-------+-------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql> insert into test2 values (50.123);
mysql> insert into test2 values (40.79);
mysql> insert into test2 values (32.41);

mysql> select 50.123 + 40.79 + 32.41;
+------------------------+
| 50.123 + 40.79 + 32.41 |
+------------------------+
|                123.323 |
+------------------------+
1 row in set (0.00 sec)

mysql> select sum(value) from test2;
+--------------------+
| sum(value)         |
+--------------------+
| 123.32300186157227 |
+--------------------+
1 row in set (0.00 sec)

mysql> select if((select (select sum(value) from test2) = 123.323),sleep(2),1);
+------------------------------------------------------------------+
| if((select (select sum(value) from test2) = 123.323),sleep(2),1) |
+------------------------------------------------------------------+
|                                                                1 |
+------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select if((select (select sum(value) from test2) > 123.323),sleep(2),1);
+------------------------------------------------------------------+
| if((select (select sum(value) from test2) > 123.323),sleep(2),1) |
+------------------------------------------------------------------+
|                                                                0 |
+------------------------------------------------------------------+
1 row in set (2.01 sec)

```

#### 浮点数和定点数的比较
浮点数如果不写经度和标度，会按照实际精度值保存，如果有精度和标度，则会自动将四舍五入后的结果插入，系统不会报错；定点数如果不写精度和标度，则按照默认值decimal(10,0) 来操作，如果数据超过了精度和标度值，系统会报错
