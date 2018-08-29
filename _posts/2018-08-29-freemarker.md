# 数据模型

## Hash类型数据模型

- 下图中的变量扮演目录的角色(比如 `root`, `animals`, `mouse`, `elephant`, `python`, `misc`) 被称为 **hashes**
- 哈希表存储其他变量(被称为子变量)， 它们可以通过名称来查找(比如 `animals`, `mouse` 或 `price`)
- 存储单值的变量 (`size`, `price`, `message` 和 `foo`) 称为 **scalars** (标量)

如果要在模板中使用子变量， 应该从root开始指定路径，每级之间用点来分隔。例如`animals.mouse.price`

```txt
(root)
  |
  +- animals
  |   |
  |   +- mouse
  |   |   |   
  |   |   +- size = "small"
  |   |   |   
  |   |   +- price = 50
  |   |
  |   +- elephant
  |   |   |   
  |   |   +- size = "large"
  |   |   |   
  |   |   +- price = 5000
  |   |
  |   +- python
  |       |   
  |       +- size = "medium"
  |       |   
  |       +- price = 4999
  |
  +- message = "It is a test"
  |
  +- misc
      |
      +- foo = "Something"
```

## Sequences类型数据模型

要访问序列的子变量，可以使用方括号形式的数字索引下标。 索引下标从0开始。要得到第一个动物的名称的话，可以这么来写代码 `animals[0].name`。

```txt
(root)
  |
  +- animals
     |
     +- (1st)
     |   |
     |   +- name = "mouse"
     |   |
     |   +- size = "small"
     |   |
     |   +- price = 50
     |
     +- (2nd)
         |
         +- name = "elephant"
         |
         +- size = "large"
         |
         +- price = 5000

```

## 总结

- 数据模型可以被看成是树形结构。
- 标量用于存储单一的值。这种类型的值可以是字符串，数字，日期/时间或者是布尔值。
- 哈希表是一种存储变量及其相关且有唯一标识名称的容器。
- 序列是存储有序变量的容器。存储的变量可以通过数字索引来检索，索引通常从0开始。

# 注释

- 叹号  `<!-- 注释 -->`【发布之后，客户端可以看到注释内容】
- 井号 `<#-- 注释 -->` 【发布之后，客户端看不到注释内容】

# 类型分类

## 标量

最基本的最简单的值

> 字符串、数字、Boolean、日期

## 容器

> 散列、序列、集合

## 子程序

> 方法和函数、自定义指令

## 其它

> 节点变量表示树结构中的节点，主要用于XML处理
>
> 标记输出一个存储已经以输出标记格式（如HTML，XML，RTF等）的文本的值，因此不能自动转义

# 语法

## 基本指令

### ${*...*} 

```html
<body>
  <h1>Welcome ${user}!</h1>
  <p>Our latest product:
  <a href="${latestProduct.url}">${latestProduct.name}</a>!
</body>
```

### if指令

- 比较运算

```txt
<#if animals.python.price < animals.elephant.price>
  蟒蛇比今天的大象便宜。
<#elseif animals.elephant.price < animals.python.price>
  今天大象比蟒蛇便宜。
<#else>
  今天大象和蟒蛇的价格相同。
</#if>
```

- boolean类型判定

```txt
<#if animals.python.protected>
  大象是保护动物!
</#if>
```

### list指令

```txt
<#--遍历animals方式一，若animals为空时，会存在一个<ul></ul> -->
<ul>
<#list animals as animal>
	<li>${animal.name} ${animal.price}</li>
</#list> 
</ul>

<#--遍历animals方式二，若animals为空时，不会存在一个<ul></ul>，即animals为空时不会进入此循环-->
<#list animals>
<ul>
    <#items as animal>
      <li>${animal}
    </#items>
</ul>
</#list>
```

### sep指令

```txt
<#--在循环中使用sep添加分隔符，sep在最后一次不会添加-->
<#list animals as animal>
	${animal.name}<#sep>, </#sep>
</#list> 
相当于animals?join(',')
```

### include指令

copyright_footer.html

```txt
<hr>
<i>
	Copyright (c) 2000 <a href="http://www.kun.com">Kun Inc</a>,
	<br>
	All Rights Reserved.
</i>
```

包含copyright_footer.html

```txt
<!DOCTYPE html>
<html>
<body>
  <p>HaHaHa...
  <#include "/copyright_footer.html">
</body>
</html>
```

## 使用内置插件

所谓的内置函数就像是方法，它们不是来自数据模型，而是由FreeMarker添加到值中。

- `user?upper_case`字符串转大写
- `user?cap_first`字符串首字母转大写
- `user?length`查询字符串长度
- `animals?size`查询List长度
- 如果你介于`<#list listName as item>`和相应的 `</#list>`标签之间：
  - `item?index`给出基于0的内部索引
  - `item?counter`给出基于1的内部索引
  - `item?item_parity`根据当前的计数器奇偶校验，返回字符串“odd ”或“even ”。

一些内置函数需要参数来指定更多的行为，例如：

- `animal.protected?string("Y", "N")`根据 `animal.protected`， 返回字符串“Y”或“N”
- `animal?item_cycle('lightRow', 'darkRow')`是`item_parity`早期更通用的变种 
- `fruits?join(", ")`使用`,`连接数组
- `user?starts_with("J")`根据当前首字母是否为`J`判断返回boolean

内置应用程序可以链接书写例如`fruits?join(", ")?upper_case `将数组使用`,`连接并转化为大写

## 处理缺失变量

使用`!`标记缺省值

```txt
<h1>Welcome ${user!"visitor"}</h1>
```

配合`if`判断是否存在此变量

```txt
<#if user??>
	<h1>Welcome ${user}</h1>
</#if> 
```

## 表达式

- 直接指定值

  - 字符串

    | 逃脱序列 | 含义             |
    | -------- | ---------------- |
    | `\"`     | 引号             |
    | `\'`     | 撇号             |
    | `\{`     | 打开大括号： `{` |
    | `\=`     | 等于字符         |
    | `\\`     | 反斜杠           |
    | `\n`     | 换行             |
    | `\r`     | 回车             |
    | `\t`     | 水平制表         |
    | `\b`     | 退格             |
    | `\f`     | 换页             |
    | `\l`     | 小于标志         |
    | `\g`     | 大于号           |
    | `\a`     | 符号： `&`       |
    | `\xCode` | 以十六进制表示   |

    不转义打印`${r"C：\foo\bar"} `

  - 数字

    > 08，+8，8.00和8是完全等效的。因此，${08}，${+8}，${8.00}并且 ${8}将所有打印完全相同

  - Boolean

    > 编写的布尔值true 或false不要使用引号

  - 序列

    指定文字序列用逗号分隔，并将整个列表放入方括号中。例如：

    ```txt
    <#list ["foo", "bar", "baz"] as x>
    	${x}
    </#list>
    ```

  - <a  name="range">范围</a>

    - `start..end`：包含结束的范围 ，不允许越界
    - `start..<end` 和 `start..!end`：不包含结束的范围 ，不允许越界
    - `start..*length`：长度限制范围 ，允许越界

  - 哈希

    指定哈希用逗号分隔，键值用`:`分隔并将整个列表放入`{}`中。例如：

    ```txt
    { "name":"green mouse", "price":150 }
    ```

- 检索变量

  - 顶级变量 ${user}
  - 哈希中检索数据`book.author.name`或者`book["author"].name` 
  - 从序列中检索数据`animals[0].name `

- 字符串操作

  - 字符串连接

    ```txt
    <#assign s = "Hello ${user}!">
    或
    <#assign s = "Hello " + user + "!">
    ```

  - 获得一个字符，和使用字符数组方式一样

    ```txt
    ${user[0]}
    ${user[4]}
    ```

  - 字符串子串，和使用序列的方式一样

    ```txt
    <#assign s = "ABCDEF">
    ${s[2..3]}		==> CD
    ${s[2..<4]}		==> CD
    ${s[2..*3]}		==> CDE
    ${s[2..*100]}	==> CDEF	
    ${s[2..]}		==> CDEF
    ```

- 序列操作

  - 拼接序列

    ```txt
    <#list ["Joe", "Fred"] + ["Julia", "Kate"] as user>
      ${user}
    </#list>
    ```

  - 序列切分<a href="#range">详见范围部分</a>

- 哈希操作

  拼接操作，以右侧优先

  ```txt
  <#assign ages = {"Joe":23, "Fred":25} + {"Joe":30, "Julia":18}>
  - Joe is ${ages.Joe}	==>- Joe is 30
  - Fred is ${ages.Fred}	==>- Fred is 25
  - Julia is ${ages.Julia}==>- Julia is 18
  ```

- 逻辑运算

  判断时添加`（）`比较直接如 `<#if (x > y)>` 

- 

