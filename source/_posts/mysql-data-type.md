---
title: MySQL(MariaDB) 数据类型
date: 2017-10-06 16:05:36
category: MySQL(MariaDB)
tags:
	- MySQL
	- MariaDB
	- 数据类型
	- data type
---

**MySQL 数据类型**

**MySQL 支持多种数据类型**

* 数值类型
* 日期/时间类型
* 字符串（字符）类型

**选择正确的数据类型对于获得高性能至关重要，三大原则：**

1. 更小的通常更好，尽量使用可正确存储数据的最小数据类型
2. 简单就好，简单数据类型的操作通常需要更少的 CPU 运算周期
3. 尽量避免 NULL  ,包含为 NULL 的列，对 MySQL 更难优化



**整型**

1. `tinyint(m)` 1 个字节 范围（-128~127)	
2. `smallint(m)` 2 个字节 范围 （-32768~32767）
3. `mediumint(m)` 3 字节 范围 （-8388608~8388607）
4. `int(m)` 4个字节 范围 （-2147483648~2147483647）
5. `bigint(m)` 8个字节 范围 (+-9.22*10 的 18 次方)

`取值范围如果加了 UNSIGNED ,最大值将翻倍，如 tinyint unsigned 的取值范围为 0~255`

`int(m) 里的 m 表示 SELECT 查询结果的显示宽度，并不影响实际的取值范围`

MySQL 可以为整型类型指定宽度，例如 INT(10)，对大多数应用来说这是没有意义的，因为它不会限制值的合法范围，只是规定了 MySQL 的一些交互工具（例如 MySQL 命令行客户端）用来显示字符的个数。对于存储和计算来说 INT(1) 和 INT(20) 是相同的。

**浮点型**

`float` 和 `double` ,也叫近似值

`float(m,d)` 单精度浮点型 8 位精度（4 个字节）m 总个数，d 小数位数

`double(m,d)` 双精度浮点型 16 位精度 ( 8 个字节) m 总个数，d 小数位数

设定一个字段，定义为 `float(6,3)`,如果插入一个数 123.45678，实际数据库里存的是 123.456，但总个数还以实际为准，即 6 位。

**定点数**

在数据库中存放的是精确值，以十进制格式存储

`decimal(m,d)` 参数 m<65 是总个数，d<30 且 d<m 是小数位

`MySQL 5.0` 和更高版本将数值打包保存到一个二进制字符串中（每 4 个字节存 9 个数字）

例如：

```mysql
decimal(18,9)
```

小数位两边将各存储 9 个数字，一共使用 9 个字节，小数点前的数字用 4 个字节，小数点后的数字使用 4 个字节，小数点本身占 1 个字节。

浮点型在存储同样范围的值时，通常比 `decimal` 使用更少的空间。`float` 使用 4 个字节存储，`doubel` 使用 8 个字节。

因为需要额外的空间和计算开销，所以应该尽量只在对小数进行精确计算是才使用 `decimal` ；比如存储财务数据。但在数据量比较大的时候，可以考虑使用 `bigint` 代替 `decimal` 。

**字符串**

`char(n)` 固定长度，最多 255 个字符

`varchar(n)` 可变长度，最多 65535 个字符

`tinytext` 可变长度，最多 255 个字符

`text` 可变长度，最多 65535 个字符

`mediumtext` 可变长度，最多 2 的 24 次方 -1 个字符

`longtext` 可变长度，最多 2 的 32 次方 -1 个字符

`binary(m)` 固定长度，可存二进制或字符，允许长度为 0~m 个字节

`varbinary(m)` 可变长度，可存二进制或字符，允许长度为 0~m 个字节

`enum(set)` 枚举，set 为一个集合



**`char` 和 `varchar`**

1. `char(n)`  若存入的字符数小于 n，则以空格补齐，查询时再将空格去掉。所以 `char` 类型存储的字符串末尾不能有空格；`varchar` 不限于此。
2. `char(n)` 固定长度，`char(4)` 不管是存入的几个字符，都将占用 4 个字节，`varchar` 是存入的实际字符数 +1 个字节，所以 `varchar(4)` ，存入 3 个字符将占用 4 个字节。
3. `char` 类型的字符串检索速度要比 `varchar` 类型的快。



**varchar 和 text**

1. `varchar` 可指定 n，`text` 不能指定，内部存储 `varchar` 是存入的实际字符数 +1 个字节，`text` 是实际字符数 +2 个字节。
2. `text` 类型不能有默认值
3. `varchar` 可直接创建索引，`text` 创建索引要指定前多少个字符。`varchar` 查询速度快于 `text`



**二进制数据 BLOB**

1. `BLOB` 和 `TEXT` 存储方式不同，`TEXT` 以文本方式存储，英文存储区分大小写，而 `BLOB` 是以二进制方式存储，不区分大小写。
2. `BLOB` 存储的数据只能整体读出。
3. `TEXT` 可以指定字符集，`BLOB` 不能指定字符集。



**日期时间类型**

1. `date` 日期 '2008-08-08'
2. `time` 时间 '12:59:59'
3. `datetime` 日期时间'2008-08-08 12:59:59'
4. `timestamp` 自动存储记录修改时间
5. `year(2)` ,`year(4)` 年份（两位数表示与四位数表示）




END!