# 一、简介

Thymeleaf是一个适用于Web和独立环境的现代服务器端Java模板引擎，能够处理HTML，XML，JavaScript，CSS甚至纯文本。

Thymeleaf具有以下几种模版模式
- HTML——标记模版模式
- XML——标记模版模式
- TXT——文本模式
-  JAVASCRIPT——文本模式
-  CSS——文本模式
-  RAW——无操作模式

**HTML**模式将允许任何类型的HTML的输入，不会执行验证或格式检查，并且尽可能尊重模板代码结构

**XML**模式将允许XML输入，在这种情况下，会进行格式检查，不会执行任何代码检查（针对DTD或Schema）

**TEXT**模式将允许非标记性质的模板使用特殊的语法。此类模板的示例可能是文本电子邮件或模板文档。HTML或XML模板也可以被处理TEXT，在这种情况下，它们不会被解析为标记，并且每个标记，DOCTYPE，注释等都将被视为纯文本

**JAVASCRIPT**能够以与HTML文件相同的方式在JavaScript文件中使用模型数据，但是使用特定于JavaScript的集成，例如专门的转义或脚本。

**CSS**与`JAVASCRIPT`模式类似，CSS模板模式也是文本模式，并使用`TEXT`模板模式中的特殊处理语法

**RAW**模式将不处理模板。它用于将未经处理的资源（文件，URL响应等）插入到正在处理的模板中

# 二、使用文本

## 多语言

Thymeleaf国际化文本位置是可配置的，它取决于`IMessageResolver`的具体实现。通常使用`.properties`文件实现。如果要从数据库获取消息，我们可以创建自己的实现。

初始化期间没有为模板引擎指定消息解析器默认使用`StandardMessageResolver`。

`StandardMessageResolver`期望在html同目录下找到与此html文件名称对应的国际化文件。如`/WEB-INF/templates/home.html`下期望找到：

- `/WEB-INF/templates/home_en.properties` 英文文本
- `/WEB-INF/templates/home_zh.properties` 中文文本
- `/WEB-INF/templates/home.properties` 默认文本（如果区域设置不匹配）

使用`#{..}`从properties文件中取值

## 上下文

Thymeleaf上下文是实现`org.thymeleaf.context.IContext`接口的对象。上下文应包含变量映射中执行模板引擎所需的所有数据，并且还引用国际化消息的语言环境。

```java
public interface IContext {
    public Locale getLocale();
    public boolean containsVariable(final String name);
    public Set<String> getVariableNames();
    public Object getVariable(final String name);
}
```

`org.thymeleaf.context.IWebContext`扩展自`IContext`用于Web应用程序（如SpringMVC）

```java
public interface IWebContext extends IContext {
    public HttpServletRequest getRequest();
    public HttpServletResponse getResponse();
    public HttpSession getSession();
    public ServletContext getServletContext();
}
```

可以使用指定的表达式从`WebContext`模板中获取请求参数以及Request，Session和ApplicationContext属性。例如：

- `${x}`将返回`x`存储在Thymeleaf上下文中的变量或作为Request属性
- `${param.x}`将返回一个请求参数x（可能多值），例如**${param.userId}**取出请求参数userId
- `${session.x}`将返回Session会话属性`x`，例如**${session.user.name}**取出session中存放的用户名
- `${application.x}`将返回Application属性`x`，例如**${application['javax.servlet.context.tempdir']}**

## 文字转义

- th:utext：不转义，使用原样输出
- th:text：转义

# 三、标准表达式语法

以下这些功能都可以组合和嵌套

- 简单表达式
  - 变量表达式：${...}
  - 选择变量表达式：*{...}
  - 消息表达式：#{...}
  - 链接网址表达式：@{...}
  - 片段表达式： ~{...}
- 字面值
  - 文本文字：'one text'，'Another one!'，...
  - 号码文字：0，34，3.0，12.3，...
  - 布尔文字：true，false
  - 空字面： null
  - 文字标记：one，sometext，main，...
- 文字操作
  - 字符串连接：+
  - 文字替换：|The name is ${name}|
- 算术运算
  - 二元运算符：+，-，*，/，%
  - 负号（一元运算符）：-
- 布尔运算
  - 二元运算符：and，or
  - 否定（一元运算符）：！，not
- 比较运算
  - 大小比较：>，<，>=，<=（gt，lt，ge，le）
  - 相等比较：==，!=（eq，ne）
- 条件操作
  - IF-THEN：(if)?(then)
  - IF-THEN-ELSE：(if)?(then):(else)
  - DEFAULT：(value)?:(defaultvalue)
- 特殊操作
  - 不操作：_

## 消息

利用[`java.text.MessageFormat`](https://docs.oracle.com/javase/10/docs/api/java/text/MessageFormat.html)标准语法进行格式化输出

```properties
home.welcome=Welcome to our grocery store, {0} (from default messages)!
```

```html
<p th:utext="#{home.welcome(${session.user.name})}">
  Welcome to our grocery store, Sebastian Pepper!
</p>
```

消息密钥本身可以来自变量

```html
<p th:utext="#{${welcomeMsgKey}(${session.user.name})}">
  Welcome to our grocery store, Sebastian Pepper!
</p>
```

## 变量

${...}实际上是OGNL表达式

```html
<p>Today is: <span th:text="${today}">13 february 2011</span>.</p
```

相当于

```java
ctx.getVariable("today");
```

```html
<p th:utext="#{home.welcome(${session.user.name})}">
  Welcome to our grocery store, Sebastian Pepper!
</p>
```

相当于

```java
((User) ctx.getVariable("session").get("user")).getName();
```

## 表达式内置基本对象

- `#ctx`：上下文对象
- `#vars:` 上下文变量
- `#locale`：上下文区域设置
- `#request`:(仅限Web Contexts）`HttpServletRequest`对象
- `#response`:(仅限Web Contexts）`HttpServletResponse`对象
- `#session`:(仅限Web Contexts）`HttpSession`对象
- `#servletContext`:(仅限Web Contexts）`ServletContext`对象

使用`${#基本对象}`取出值，例如${#locale`}

## 表达式内置工具对象

- `#execInfo`：有关正在处理的模板的信息
- `#messages`：在变量表达式中获取外化消息的方法，与使用`＃{...}`语法获取的方法相同
- `#uris`：转义部分URL / URI的方法
- `#conversions`：用于执行已配置的转换服务的方法（如果有）
- `#dates`：`java.util.Date`对象的方法：格式化，时间提取等
- `#calendars`：`java.util.Calendar`对象
- `#numbers`：格式化数字对象的方法
- `#strings`：`String`对象的方法：contains，startsWith，prepending / appending等
- `#objects`：一般的对象的方法
- `#bools`：布尔方法
- `#arrays`：数组方法
- `#lists`：列表方法
- `#sets`：集合方法
- `#maps`：哈希方法
- `#aggregates`：在数组或集合上创建聚合的方法。
- `#ids`：处理可能重复的id属性的方法（例如，作为迭代的结果）

例如处理时间格式：

```html
<p>
  Today is: 
  <span th:text="${#dates.format(today,'yyyy-MM-dd')}">2018-08-22</span>
</p>
```

## 选择表达式

`*{..}`相当于从上层的`th:object="${..}"`取值

```html
<div th:object="${session.user}">
  <p>Name: <span th:text="*{firstName}">Sebastian</span>.</p>
  <p>Surname: <span th:text="*{lastName}">Pepper</span>.</p>
  <p>Nationality: <span th:text="*{nationality}">Saturn</span>.</p>
</div>
```

相当于

```html
<div>
  <p>Name: <span th:text="${session.user.firstName}">Sebastian</span>.</p>
  <p>Surname: <span th:text="${session.user.lastName}">Pepper</span>.</p>
  <p>Nationality: <span th:text="${session.user.nationality}">Saturn</span>.</p>
</div>
```

## 链接表达式

使用`@{...}`处理链接

`http://localhost/thymeleaf/order/details?orderId=15 `有以下几种写法

```html
<a href="details.html" 
   th:href="@{http://localhost:8080/gtvg/order/details(orderId=${o.id})}">view</a>
```

相当于

```html
<a href="details.html" th:href="@{/order/details(orderId=${o.id})}">view</a>
```

相当于

```html
<a href="details.html" th:href="@{/order/details(orderId=${o.id})}">view</a>
```

RESTFULL风格写法：`http://localhost/thymeleaf/order/15/details`

```html
<a href="details.html" th:href="@{/order/{orderId}/details(orderId=${o.id})}">view</a>
```

注意RESTFULL风格的`@{...}`可变orderId不加`$`

## 段表达式

[详见模版布局](#templateLayout)

## 字面值

### 文本

```html
<p>
  Now you are looking at a
  <span th:text="'working web application'">template file</span>.
</p>
```

### 数字

```html
<p>The year is <span th:text="2013">1492</span>.</p>
<p>In two years, it will be <span th:text="2013 + 2">1494</span>.</p>
```

### 布尔值判断

==false在大括号后面有thymeleaf处理

```html
<div th:if="${user.isAdmin()} == false"> ...
```

==false在大括号里面由ONGL/SpringEL引擎负责

```html
<div th:if="${user.isAdmin() == false}"> ...
```

### null值判断

```html
<div th:if="${variable.something} == null">
```

### 文字替代

```html
<div th:class="content">...</div>
相当于
<div th:class="'content'">...</div>
```

## 拼接文本

使用`+`拼接

```html
<span th:text="'The name of the user is ' + ${user.name}"></span>
```

相当于

```html
<span th:text="|Welcome to our application, ${user.name}!|"></span>
```

## 算数运算

可以使用`+`， `-`，`*`， `/` ， `%`进行运算

```html
<!-- thymeleaf处理 -->
<div th:with="isEven=(${prodStat.count} % 2 == 0)">
```

结果相当于

```html
<!-- ONGL、SpEL处理 -->
<div th:with="isEven=${prodStat.count % 2 == 0}">
```

## 等于和大小判断

可以使用`>`，`<`，`>=`和`<=`符号，以及`==`和`!=`进行比较运算

```html
<div th:if="${prodStat.count} > 1">
<span th:text="((${execMode} == 'dev')? 'Development' : 'Production')">
```

可以使用`&lt;`（`<`）和`&gt`（`>`）

或者使用别名`gt` (`>`), `lt` (`<`), `ge` (`>=`), `le` (`<=`)

## 条件表达式

使用`？:`进行三元运算

```html
<tr th:class="${row.even}? 'even' : 'odd'">
  ...
</tr>
```

表达式也可以省略部分

```html
<!-- 返回值为false时，返回null,显示<span></span> -->
<span th:text="${row.even}? 'alt'"></span>
```

```html
<!-- 返回值为true时，返回'true',显示<span>true</span> -->
<span th:text="${row.even}? :'alt'"></span>
```

## 默认表达式（Elvis运算符）

```html
<div th:object="${session.user}">
  <p>Age: <span th:text="*{age}?: '(no age specified)'">27</span>.</p>
</div>
```

上述代码相当于age为空时设置默认值，与下面代码作用相同

```html
<p>Age: <span th:text="*{age != null}? *{age} : '(no age specified)'">27</span>.</p>
```

## 无操作表达式

```html
<span th:text="${user.name} ?: 'no user authenticated'">...</span>
```

相当于

```html
<span th:text="${user.name} ?: _">no user authenticated</span>
```

## 数据转换/格式化服务

见更多配置部分

`${{...}}`和`*{{...}}`

```html
<td th:text="${{user.lastAccessDate}}">...</td>
```

先获得user.lastAccessDate的值，在调用注册的转换服务，如果没有仅仅会调用toString方法

> thymeleaf-spring3和thymeleaf-spring4集成软件包的集成了Spring与Thymeleaf的转换服务机制，所以在Spring配置宣称，转换服务和格式化将进行自动获得`${{...}}`和`*{{...}}`表达

##  预处理

预处理表达式与普通表达式完全相同，但显示为双下划线符号（如`__${expression}__`、`__#{expression}__`...）

例如

Messages_zh.properties

```properties
article.text=@myapp.translator.Translator@translateToChinese({0})
```

Messages_en.properties

```properties
article.text=@myapp.translator.Translator@translateToEnglish({0})
```

使用预处理

```html
<p th:text="${__#{article.text('textVar')}__}">Some text here...</p>
```

更具语言环境会被替换为

```html
<p th:text="${@myapp.translator.Translator@translateToChinese(textVar)}"></p>
或者
<p th:text="${@myapp.translator.Translator@translateToEnglish(textVar)}"></p>
```

# 四、设置属性值

## 设置任何属性的值

使用`th:attr`设置任何属性

```html
<img src="/images/gtvglogo.png" 
     th:attr="src=@{/images/gtvglogo.png},title=#{logo},alt=#{logo}" />
```

处理之后的结果为

```html
<img src="/gtgv/images/gtvglogo.png"
     title="Logo de Good Thymes"
     alt="Logo de Good Thymes" />
```

## 为指定属性设置值

```html
<img src="/images/gtvglogo.png" 
     th:attr="src=@{/images/gtvglogo.png},title=#{logo},alt=#{logo}" />
```

相当于

```html
<img src="/images/gtvglogo.png" 
     th:src="@{/images/gtvglogo.png}"
     th:title="#{logo}"
     th:alt="#{logo}" />
```

## 一次设置多个值

有两个比较特殊的属性`th:alt-title`和`th:lang-xmllang`可用于同时设置两个属性相同的值：

- `th:alt-title`将设置`alt`和`title`。
- `th:lang-xmllang`将设置`lang`和`xml:lang`。

```html
<img src="/images/gtvglogo.png" 
     th:src="@{/images/gtvglogo.png}" 
     th:title="#{logo}" 
     th:alt="#{logo}" />
```

相当于

```html
<img src="/images/gtvglogo.png" 
     th:src="@{/images/gtvglogo.png}" th:alt-title="#{logo}" />
```

## 前后添加属性

```html
<p class="blockfont" th:attrappend="class=${' ' + cssStyle}">Hello</p>
```

拼接结果

```html
<p class="blockfont redfont">Hello</p>
```

```html
<p class="blockfont" th:attrprepend="class=${cssStyle + ' '}">Hello</p>
```

拼接结果

```html
<p class="redfont blockfont">Hello</p>
```

标准方言中还有两个属性：`th:classappend`和`th:styleappend`，用于向元素添加CSS类或Style片段而不覆盖现有元素

## 固定值布尔属性

标准方言将计算表达式的值，如果为true，则将属性设置为其固定值，如果计算为false，则不会设置该属性

```html
<input type="checkbox" name="active" th:checked="${user.active}" />
```

## 设置任何属性的值

Thymeleaf提供了一个默认属性处理器，允许设置任何属性的值，即使`th:*`，在标准方言中没有为它定义特定的处理器

```html
<span th:whatever="${user.name}">...</span>
```

结果为

```html
<span whatever="John Apricot">...</span>
```

## HTML5友好的属性和元素名称

也可以使用HTML5规定的用户自定义属性写法`data-{prefix}-{name}`,此做法无需开使用任何命名空间的名称

例如`<span data-th-text="${user.login}">...</span>`

# 五、循环

## 基础

使用`th:each`实现遍历

- list遍历

  ```html
  <ul th:each="user : ${users}">
      <li th:text="${user.firstName}"></li>
  </ul>
  ```

- map遍历

  ```html
  <ul th:each="user : ${userMaps}">
      <li th:text="|${user.key} - ${user.value.firstName}"></li>
  </ul>
  ```

## 迭代状态

状态变量在`th:each`属性中定义，包含以下属性：

|   属性   |          描述          |
| :------: | :--------------------: |
|  index   | 当前迭代索引，从0开始  |
|  count   | 当前迭代索引，从1开始  |
|   size   |    集合中元素的总量    |
| current  |     当前遍历的元素     |
| even/odd | 当前迭代是偶数还是奇数 |
|  first   |  当前迭代是否是第一个  |
|   last   | 当前迭代是否是最后一个 |

使用`迭代变量,迭代状态变量:集合`的方式进行迭代，如果没有显示指定迭代变量，则thymeleaf创建以迭代变量名+`Stat`命令的迭代状态变量，例如

```html
<ul th:each="user,iterStat : ${users}">
    <li th:text="${user.firstName}"></li>
    <li th:text="${iterStat.first}"></li>
    <li th:text="${iterStat.last}"></li>
    <li th:text="${iterStat.index}"></li>
    <li th:text="${iterStat.count}"></li>
    <li th:text="${iterStat.size}"></li>
    <li th:text="${iterStat.current}"></li>
    <li th:text="${iterStat.even}"></li>
    <li th:text="${iterStat.odd}"></li>
</ul>
```

相当于

```html
<ul th:each="user : ${users}">
    <li th:text="${user.firstName}"></li>
    <li th:text="${userStat.first}"></li>
    <li th:text="${userStat.last}"></li>
    <li th:text="${userStat.index}"></li>
    <li th:text="${userStat.count}"></li>
    <li th:text="${userStat.size}"></li>
    <li th:text="${userStat.current}"></li>
    <li th:text="${userStat.even}"></li>
    <li th:text="${userStat.odd}"></li>
</ul>
```

## 懒加载

设置变量时`public void setVariable(final String name, final Object value)`,被放入上下文环境中的类实现`LazyContextVariable`接口即可实现懒加载，如果加载的条件不满足，则不会触发`loadValue`方法。注意**每次加载都会触发loadValue**

```java
 ctx.setVariable(
	"users",
	new LazyContextVariable<List<User>>() {
    	@Override
        protected List<User> loadValue() {
        	return users.findAllUsers();
        }
    })
```

# 六、条件评估

`th:if`属性按照true以下规则评估指定的表达式：

- 如果value不为null：
  - 如果value是布尔值，则为`true`。
  - 如果value是数字且不为零
  - 如果value是一个字符且不为零
  - 如果value是String并且不是“false”，“off”或“no”
  - 如果value不是布尔值，数字，字符或字符串。
- 如果value为null，则为false

`unless`相当于"if not"

***

`switch`语句，默认值为`*`

```html
<div th:switch="${user.role}">
  <p th:case="'admin'">User is an administrator</p>
  <p th:case="#{roles.manager}">User is a manager</p>
  <p th:case="*">User is some other thing</p>
</div>
```

# <span name="templateLayout">七、模版布局</span>

## 定义模版片段

使用`th:fragment`定义片段，例如：

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
  <body>
    <div th:fragment="copy">
      &copy; 2011 The Good Thymes Virtual Grocery
    </div>
  </body>
</html>
```

## 引用片段

- 使用`th:insert`插入引用片段

  ```html
  <body>
    <div th:insert="~{footer :: copy}"></div>
  </body>
  ```

  结果

  ```html
  <body>
      <div>
          <div>
            &copy; 2011 The Good Thymes Virtual Grocery
          </div>
      </div>
  <body>
  ```

- 使用`th:replace`插入引用片段

  ```html
  <body>
    <div th:insert="~{footer :: copy}"></div>
  </body>
  ```

  结果

  ```html
  <body> 
  	<div>
          &copy; 2011 The Good Thymes Virtual Grocery
      </div>
  </body>
  ```


## 片段规范语法

- 使用`th:insert="~{文件名::片段名}"`或者`th:insert="文件名::片段名"`
- `"~{templatename}"`包含名为`templatename`的完整模板，会将把指定元素替换为`templatename`整个网页
- `~{::selector}"`或`"~{this::selector}"`插入来自当前模板的片段，进行匹配`selector`。如果在表达式出现的模板上找不到，则模板调用（插入）的堆栈将遍历最初处理的模板（根），直到`selector`在某个级别匹配

## 不标记th:fragment

```html
<div id="copy-section">
  &copy; 2011 The Good Thymes Virtual Grocery
</div>
```

引用此片段（使用id选择器）

```html
<body>
  <div th:insert="~{footer :: #copy-section}"></div>
</body>
```

## 可参数化的片段签名

声明片段

```html
<div th:fragment="frag (onevar,twovar)">
    <p th:text="${onevar} + ' - ' + ${twovar}">...</p>
</div>
```

调用此片段

```html
<div th:replace="::frag (${value1},${value2})">...</div>
<div th:replace="::frag (onevar=${value1},twovar=${value2})">...</div
```

## 灵活布局

```html
<head th:fragment="common_header(title,links)">

  <!--/* 替换title */-->
  <title th:replace="${title}">The awesome application</title>

  <!-- 不是重点 -->
  <link rel="shortcut icon" th:href="@{/images/favicon.ico}">
  <script type="text/javascript" th:src="@{/sh/scripts/codebase.js}"></script>

  <!--/* 会将传入的links全部替换在此 */-->
  <th:block th:replace="${links}" />
</head>
```

调用此片段

```html
<head th:replace="base :: common_header(~{::title},~{::link})">

  <title>Awesome - Main</title>

  <link rel="stylesheet" th:href="@{/css/bootstrap.min.css}">
  <link rel="stylesheet" th:href="@{/themes/smoothness/jquery-ui.css}">
</head>
```

结果，使用的调用者本身的title，加上调用者本身的链接

```html
<head>
  <title>Awesome - Main</title>
    
  <!-- 不是重点 -->
  <link rel="shortcut icon" href="/awe/images/favicon.ico">
  <script type="text/javascript" src="/awe/sh/scripts/codebase.js"></script>

  <link rel="stylesheet" href="/awe/css/bootstrap.min.css">
  <link rel="stylesheet" href="/awe/themes/smoothness/jquery-ui.css">

</head>
```

无标记传入

```html
<head th:replace="base :: common_header(~{::title},~{})">

  <title>Awesome - Main</title>

</head>
```

结果为

```html
<head>

  <title>Awesome - Main</title>

  <!-- 不是重点 -->
  <link rel="shortcut icon" href="/awe/images/favicon.ico">
  <script type="text/javascript" src="/awe/sh/scripts/codebase.js"></script>

</head>
```

无操作`_`标记传入

```html
<head th:replace="base :: common_header(_,~{::link})">

  <title>Awesome - Main</title>

  <link rel="stylesheet" th:href="@{/css/bootstrap.min.css}">
  <link rel="stylesheet" th:href="@{/themes/smoothness/jquery-ui.css}">

</head>
```

结果为

```html
<head>

  <title>The awesome application</title>

  <!-- 不是重点 -->
  <link rel="shortcut icon" href="/awe/images/favicon.ico">
  <script type="text/javascript" src="/awe/sh/scripts/codebase.js"></script>

  <link rel="stylesheet" href="/awe/css/bootstrap.min.css">
  <link rel="stylesheet" href="/awe/themes/smoothness/jquery-ui.css">

</head>
```

## 删除模板片段

模拟数据的时候会模拟很多行，那么循环遍历是就需要删除不需要的数据例如

```html
<table>
  <thead>
    <tr>
      <th>姓名</th>
      <th>年龄</th>
    </tr>
    <tr>
      <td>小明</td>
      <td>18</td>
    </tr>
    <tr>
      <td>小红</td>
      <td>16</td>
    </tr>
	<tr>
      <td>小白</td>
      <td>20</td>
    </tr>
  </thead>
</table>
```

使用`th:remove`处理

```html
<table>
  <thead>
    <tr>
      <th>姓名</th>
      <th>年龄</th>
    </tr>
    <tr th:each="user : ${users}" th:remove="all-but-first">
      <td>[[user.name]]</td>
      <td>[[user.age]]</td>
    </tr>
    <tr>
      <td>小明</td>
      <td>18</td>
    </tr>
    <tr>
      <td>小红</td>
      <td>16</td>
    </tr>
	<tr>
      <td>小白</td>
      <td>20</td>
    </tr>
  </thead>
</table>
```

`th:remove`可选属性

- `all`：删除包含标记及其所有子标记。
- `body`：不要删除包含标记，但删除其所有子标记。
- `tag`：删除包含标记，但不删除其子标记。
- `all-but-first`：除第一个子项外，删除包含标记的所有子项。
- `none`： 没做什么。

# 八、局部变量

hymeleaf提供了一种使用`th:with`属性声明局部变量方法，可以使用`,`定义多个，其语法类似于属性值赋值：

```html
<!-- firstPer只在声明的div块内可见 -->
<div th:with="firstPer=${persons[0]}">
  <p>
     <span th:text="${firstPer.name}"></span>
  </p>
</div>
```

场景：如果根据地区不同显示不同的日期格式，可以使用下述方法

home_en.properties：

```properties
date.format=MM dd yyyy
```

home_zh.properties：

```properties
date.format=yyyy-MM-dd
```

`th:with`使用

```html
<p th:with="df=#{date.format}">
  Today is: <span th:text="${#date.format(today,df)}"></span>
</p
```

或者

```html
<p>
  Today is: 
  <span th:with="df=#{date.format}" th:text="${#date.format(today,df)}"></span>
</p>
```

# 九、属性优先级

| 优先级别 |     描述     |      属性      |
| :------: | :----------: | :------------: |
|    1     |   片段包含   |   th:insert    |
|          |              |   th:replace   |
|    2     |     迭代     |    th:each     |
|    3     |   条件判断   |     th:if      |
|          |              |   th:unless    |
|          |              |   th:switch    |
|          |              |    th:case     |
|    4     | 局部变量定义 |   th:object    |
|          |              |    th:with     |
|    5     | 一般属性修改 |    th:attr     |
|          |              | th:attrprepend |
|          |              | th:attrappend  |
|    6     | 具体属性修改 |    th:value    |
|          |              |    th:href     |
|          |              |     th:src     |
|          |              |      ...       |
|    7     |     文字     |    th:text     |
|          |              |    th:utext    |
|    8     |   片段定义   |  th:fragment   |
|    9     |   片段删除   |   th:remove    |

# 十、注释和块

- 标准HTML/XML注释

  标准HTML / XML注释`<!-- ... -->`可以在Thymeleaf模板中的任何位置使用。Thymeleaf不会处理这些注释，并将逐字复制到结果中

- Thymeleaf解析器级别注释

  Thymeleaf将简单删除一切与`<!--/*`和`*/-->`之间的注释，不会解析和显示

  ```html
  <!--/* 单行注释掉! */-->
  ```

  ```html
  <!--/* 
  	...
  	整块注释掉
  	...
  */-->  
  ```

- 原型的注释

  `<!--/*/    /*/-->`，thymeleaf会解析但是不会显示

- 搭配`th:block`标签

  `th:block`和`<!--/*/ /*/-->`搭配使用，`th:block`下的变量可见，不需要再写上级标签

  ```html
  <table>
      <!--/*/ <th:block th:each="user : ${users}"> /*/-->
      <tr>
          <td th:text="${user.login}">...</td>
          <td th:text="${user.name}">...</td>
      </tr>
      <!--/*/ </th:block> /*/-->
  </table>
  ```

# 十一、内联

## 内联表达式

内联表达式`th:inline="text"`默认是开启的，`th:inline="javascript"`和`th:inline="css"`需要显示开启

- `[[...]]`相当于`th:text`
- `[(...)]`相当于`th:utext`

- `th:inline="none"`禁用内联表达式

  ```html
  <p th:inline="none">A double array looks like this: [[1, 2, 3], [4, 5]]!</p>
  ```

  结果为

  ```html
  <p>A double array looks like this: [[1, 2, 3], [4, 5]]!</p>
  ```

## 文字内联

[详见文本模版](#textTemplate)

## JavaScript内联

[详见文本模版](#textTemplate)

## Css内联

```html
<style th:inline="css">
  ...
</style>
```

例如

> classname = 'main elems'
> align = 'center'

```html
<style th:inline="css">
    .[[${classname}]] {
      text-align: [[${align}]];
    }
</style>
<style th:inline="css">
    .[(${classname})] {
      text-align: [[${align}]];
    }
</style>
```

结果将是

```html
<style th:inline="css">
    .main\ elems {
      text-align: center;
    }
</style>
<style th:inline="css">
    .main elems {
      text-align: center;
    }
</style>
```

高级自然模版写法，后面声明的元素会被替换

```html
<style th:inline="css">
    .main elems {
      text-align: /*[[${align}]]*/ left;
    }
</style>
```

# <span name="textTemplate">十二、文本模板模式</span>

`TEXT`，`JAVASCRIPT`和`CSS`均属于文本模板，`HTML`和`XML`属于标记模板。

## 文本语法

文本模板模式与标记模式之间的关键区别在于，文本模板中没有**标签**可以以属性的形式插入逻辑，因此我们必须依赖其他机制。

- 可以在文本模版中使用`[[${...}]]`或者`[(...)]`进行取值操作

- 文本模版中遍历

  ```text
  [#th:block th:each="item : ${items}"]
    - [#th:block th:utext="${item}" /]
  [/th:block]
  ```

  可以使用`[# ...] ... [/]` 替代`[#th:block ...]... [/th:block]`

  ```text
  [# th:each="item : ${items}"]
    - [# th:utext="${item}" /]
  [/]
  ```

  `[# th:utext="${...}" /]`等效于内联非转义表达式，所以可以使用如下替代

  ```text
  [# th:each="item : ${items}"]
    - [(${item})]
  [/]
  ```

- 文本模版中判断

  ```text
  [# th:if="${oid}>0" ]
  	大于0
  [/]
  [# th:if="${session.user.firstName}=='小明'" ]
  	他叫小明
  [/]
  ```

## 原型注代码块

在`JAVASCRIPT`和`CSS`模板模式（不适用于`TEXT`），允许包括一个特殊的注释语法之间的代码`/*[+...+]*/`，这样Thymeleaf处理模板时会自动取消注释，将其输出

```text
var x = 23;

/*[+

var msg  = "Hello, " + [[${session.user.name}]];

+]*/
```

将被执行为

```text
var x = 23;

var msg  = "Hello,小明";
```

## 原型注释块

```text
var x = 23;

/*[+

/*[- 我是注释 -]*/
var msg  = "Hello, " + [[${session.user.name}]];

+]*/
```

## JavaScript和CSS自然模版

JavaScript和CSS内联提供了在JavaScript / CSS注释中包含内联表达式的方式

```javascript
...
var username = /*[[${session.user.name}]]*/ "Sebastian Lychee";
...
```

```javascript
/*[# th:if="${user.admin}"]*/
	alert('Welcome admin');
/*[/]*/
```

上述方式javascript可以正常打开

# 十三、有关配置信息 

## 模板解析器

Thymeleaf 自身模板解析器

```txt
ITemplateResolver
  |
  +- AbstractTemplateResolver
	 |
  	 +- DefaultTemplateResolver【默认模版解析器】
  	 |
  	 +- StringTemplateResolver【可以直接解析模板】
  	 |
  	 +- AbstractConfigurableTemplateResolver
  	 	|
  	 	+- ClassLoaderTemplateResolver【将模板解析为类加载器资源】
  	 	|
  	 	+- FileTemplateResolver【将模板解析为文件系统中的文件】
  	 	|
  	 	+- ServletContextTemplateResolver【从Servlet Context获取模板作为资源】
  	 	|
  	 	+- UrlTemplateResolver【将模板解析为URL】
```

设置相关属性

```java
//前缀和后缀
templateResolver.setPrefix("/WEB-INF/templates/");
templateResolver.setSuffix(".html");

//设置编码
templateResolver.setEncoding("UTF-8");

//要使用的模板模式,默认HTML
templateResolver.setTemplateMode("XML");

//模板缓存，默认开启
templateResolver.setCacheable(false);
templateResolver.getCacheablePatternSpec().addPattern("/users/*");

//缓存条目的TTL，如果未设置，从缓存中删除条目的唯一方法是超过缓存最大大小（将删除最旧的条目）
templateResolver.setCacheTTLMs(60000L);
```

### 链接模板解析器

模板引擎可以指定多个模板解析器，在这种情况下，可以在它们之间建立用于模板解析的顺序（通过setOrder方法），这样，如果第一个无法解析模板，则会询问第二个，依此类推，例如：

```java
ClassLoaderTemplateResolver classLoaderTemplateResolver = new ClassLoaderTemplateResolver();
classLoaderTemplateResolver.setOrder(Integer.valueOf(1));

ServletContextTemplateResolver servletContextTemplateResolver = 
        new ServletContextTemplateResolver(servletContext);
servletContextTemplateResolver.setOrder(Integer.valueOf(2));

templateEngine.addTemplateResolver(classLoaderTemplateResolver);
templateEngine.addTemplateResolver(servletContextTemplateResolver)
```

当应用多个模板解析器时，建议为每个模板解析器指定模式，以便Thymeleaf可以快速丢弃那些不打算解析模板的模板解析器，从而提高性能，例如：

```java
ClassLoaderTemplateResolver classLoaderTemplateResolver = new ClassLoaderTemplateResolver();
classLoaderTemplateResolver.setOrder(Integer.valueOf(1));
//如果路径不匹配则直接不使用此解析器 
classLoaderTemplateResolver.getResolvablePatternSpec().addPattern("/layout/*.html");
classLoaderTemplateResolver.getResolvablePatternSpec().addPattern("/menu/*.html");

ServletContextTemplateResolver servletContextTemplateResolver = 
        new ServletContextTemplateResolver(servletContext);
servletContextTemplateResolver.setOrder(Integer.valueOf(2));
```

如果未指定这些Pattern，我们将依赖于`ITemplateResolver`每个实现的特定功能【可能不是我们想要的结果】

## 消息解析器

`StandardMessageResolver`在定位当解析资源时，会在同级目录下寻找properties文件

```txt
IMessageResolver
  |
  +- AbstractMessageResolver
  	 |
  	 +- StandardMessageResolver【默认使用】
```

### 链接消息解析器

```java
//设置唯一一个
templateEngine.setMessageResolver(messageResolver);

//添加一个，如果第一个消息解析器无法解析特定消息，则会询问第二个，然后是第三个...
templateEngine.addMessageResolver(messageResolver);
...
```

## 转换服务

使用`${{...}}`调用转换服务，配置转换服务的方法：

```java
IStandardConversionService customConversionService = ...

StandardDialect dialect = new StandardDialect();
dialect.setConversionService(customConversionService);

templateEngine.setDialect(dialect);
```

## 日志‘

Thymeleaf使用的日志门面是`slf4j`，配置示例：

```properties
log4j.logger.org.thymeleaf=DEBUG
log4j.logger.org.thymeleaf.TemplateEngine.CONFIG=TRACE
log4j.logger.org.thymeleaf.TemplateEngine.TIMER=TRACE
log4j.logger.org.thymeleaf.TemplateEngine.cache.TEMPLATE_CACHE=TRACE
```

# 十四、模板缓存

启用缓存代码

```java
//默认true
templateResolver.setCacheable(true);
//设置指定的缓存模版，默认全部
templateResolver.getCacheablePatternSpec().addPattern("/users/*");
//默认200
cacheManager.setTemplateCacheMaxSize(100);
```

缓存管理器接口`ICacheManager`,默认实现类`StandardCacheManager`

# 十五、模版逻辑解耦

## 配置解耦模版

Thymeleaf可以彻底与模板逻辑脱钩，将逻辑创建在XML文件当中。

例如`home.html`文件可以完全无逻辑：

```html
<!DOCTYPE html>
<html>
  <body>
    <table id="usersTable">
      <tr>
        <td class="username">Jeremy Grapefruit</td>
        <td class="usertype">Normal User</td>
      </tr>
      <tr>
        <td class="username">Alice Watermelon</td>
        <td class="usertype">Administrator</td>
      </tr>
    </table>
  </body>
</html>
```

创建逻辑附加文件`home.th.xml`

```xml
<?xml version="1.0"?>
<thlogic>
  <attr sel="#usersTable" th:remove="all-but-first">
    <attr sel="/tr[0]" th:each="user : ${users}">
      <attr sel="td.username" th:text="${user.name}" />
      <attr sel="td.usertype" th:text="#{|user.type.${user.type}|}" />
    </attr>
  </attr>
</thlogic>
```

相当于

```html
<!DOCTYPE html>
<html>
  <body>
    <table id="usersTable" th:remove="all-but-first">
      <tr th:each="user : ${users}">
        <td class="username" th:text="${user.name}">Jeremy Grapefruit</td>
        <td class="usertype" th:text="#{|user.type.${user.type}|}">Normal User</td>
      </tr>
      <tr>
        <td class="username">Alice Watermelon</td>
        <td class="usertype">Administrator</td>
      </tr>
    </table>
  </body>
</html>
```

## 启动解耦模版

```java
ServletContextTemplateResolver templateResolver = 
        new ServletContextTemplateResolver(servletContext);
templateResolver.setUseDecoupledLogic(true);
```

## th：ref属性

`th:ref`只是一个标记属性，避免将HTML中加入大量id（锚点作用）污染文件

`th:ref`属性的适用性**不仅适用于解耦的逻辑模板文件**，它还可以用于片段表达式（`~{...}`）。

## IDecoupledTemplateLogicResolver接口

`org.thymeleaf.templateparser.markup.decoupled.IDecoupledTemplateLogicResolver`的默认实现是`StandardDecoupledTemplateLogicResolver`,它具有以下默认标准

- 默认情况下，前缀为空，后缀为`.th.xml`。

- 模板资源与原本资源具有相同名称的相对路径`ITemplateResource#relative(String relativeLocation)`

```java
StandardDecoupledTemplateLogicResolver decoupledresolver = 
        new StandardDecoupledTemplateLogicResolver();
decoupledResolver.setPrefix("../viewlogic/");
templateEngine.setDecoupledTemplateLogicResolver(decoupledResolver);
```

# 附录

## 表达式基本对象

### 基础对象

```java
//vars并且#root是同一对象的同义词，但#ctx建议使用
//IContext接口的基本对象
${#ctx.locale}
${#ctx.variableNames}

//IWebContext接口的基本对象
${#ctx.request}
${#ctx.response}
${#ctx.session}
${#ctx.servletContext}

//之间访问locale
${#locale}
```

### 请求/会话属性

```java
/*
 * =================================================================================
 * Param相关
 * =================================================================================
 */
${param.foo}              //取出foo参数
${param.size()}
${param.isEmpty()}
${param.containsKey('foo')}
/*
 * =================================================================================
 * Session相关
 * =================================================================================
 */
${session.foo}           //取出session属性foo
${session.size()}
${session.isEmpty()}
${session.containsKey('foo')}
/*
 * =================================================================================
 * Application
 * =================================================================================
 */
${application.foo}       //application
${application.size()}
${application.isEmpty()}
${application.containsKey('foo')}
```

### Web上下文对象

```java
/*
 * =================================================================================
 * HttpServletRequest
 * =================================================================================
 */
${#request.getAttribute('foo')}
${#request.getParameter('foo')}
${#request.getContextPath()}
${#request.getRequestName()}
...
/*
 * =================================================================================
 * HttpSession
 * =================================================================================
 */    
${#session.getAttribute('foo')}
${#session.id}
${#session.lastAccessedTime}
...
/*
 * =================================================================================
 * ServletContext
 * =================================================================================
 */   
${#servletContext.getAttribute('foo')}
${#servletContext.contextPath}
...
```

# 表达式内置对象 

### 执行信息

**#execInfo**：表达式对象，提供有关在Thymeleaf标准表达式中处理的模板的有用信息

```html
//当前模版名、模版模式
${#execInfo.templateName}
${#execInfo.templateMode}

//根模版名、模版模式
${#execInfo.processedTemplateName}
${#execInfo.processedTemplateMode}

//返回堆栈上模版信息
${#execInfo.templateNames}
${#execInfo.templateModes}

//返回模版栈
${#execInfo.templateStack}
```

### 消息

**#messages**：用于在变量表达式中获取国际化消息，与使用`#{...}`语法获取它们的方式相同

```html
/*
 * =================================================================================
 * message，返回默认值用法${#messages.msg('msgKey')}？：defaultValue
 * =================================================================================
 */   
${#messages.msg('msgKey')}
${#messages.msg('msgKey', param1)}
${#messages.msg('msgKey', param1, param2)}
${#messages.msg('msgKey', param1, param2, param3)}
${#messages.msgWithParams('msgKey', new Object[] {param1, param2, param3, param4})}
${#messages.arrayMsg(messageKeyArray)}
${#messages.listMsg(messageKeyList)}
${#messages.setMsg(messageKeySet)}
/*
 * =================================================================================
 * message，返回Null用法
 * =================================================================================
 */
${#messages.msgOrNull('msgKey')}
${#messages.msgOrNull('msgKey', param1)}
${#messages.msgOrNull('msgKey', param1, param2)}
${#messages.msgOrNull('msgKey', param1, param2, param3)}
${#messages.msgOrNullWithParams('msgKey', new Object[] {param1, param2, param3, param4})}
${#messages.arrayMsgOrNull(messageKeyArray)}
${#messages.listMsgOrNull(messageKeyList)}
${#messages.setMsgOrNull(messageKeySet)}
```

### URI

**#uris**：用于在Thymeleaf标准表达式中执行URI / URL操作的对象

```html
//转义/不转义作为URI/URL
${#uris.escapePath(uri)}
${#uris.escapePath(uri, encoding)}
${#uris.unescapePath(uri)}
${#uris.unescapePath(uri, encoding)}

//转义/不转义作为URI/URL路径段(在'/'符号之间)
${#uris.escapePathSegment(uri)}
${#uris.escapePathSegment(uri, encoding)}
${#uris.unescapePathSegment(uri)}
${#uris.unescapePathSegment(uri, encoding)}

//转义/不转义作为段标识
${#uris.escapeFragmentId(uri)}
${#uris.escapeFragmentId(uri, encoding)}
${#uris.unescapeFragmentId(uri)}
${#uris.unescapeFragmentId(uri, encoding)}

//转义/不转义作为查询参数
${#uris.escapeQueryParam(uri)}
${#uris.escapeQueryParam(uri, encoding)}
${#uris.unescapeQueryParam(uri)}
${#uris.unescapeQueryParam(uri, encoding)}
```

### 转换

```html
//转换对象为java.util.TimeZone
${#conversions.convert(object, 'java.util.TimeZone')}
${#conversions.convert(object, targetClass)}
```

### 日期

**#dates**：`java.util.Date`对象

```html
//转换为本地日期格式
${#dates.format(date)}
${#dates.arrayFormat(datesArray)}
${#dates.listFormat(datesList)}
${#dates.setFormat(datesSet)}

//格式日期采用ISO8601格式
${#dates.formatISO(date)}
${#dates.arrayFormatISO(datesArray)}
${#dates.listFormatISO(datesList)}
${#dates.setFormatISO(datesSet)}

//格式日期使用指定的模式
${#dates.format(date, 'dd/MMM/yyyy HH:mm')}
${#dates.arrayFormat(datesArray, 'dd/MMM/yyyy HH:mm')}
${#dates.listFormat(datesList, 'dd/MMM/yyyy HH:mm')}
${#dates.setFormat(datesSet, 'dd/MMM/yyyy HH:mm')}

//获取日期属性
${#dates.day(date)}              //arrayDay(...), listDay(...), etc.
${#dates.month(date)}            //arrayMonth(...), listMonth(...), etc.
${#dates.monthName(date)}        //arrayMonthName(...), listMonthName(...), etc.
${#dates.monthNameShort(date)}	 //arrayMonthNameShort(...), listMonthNameShort(...), etc.
${#dates.year(date)}			 //arrayYear(...), listYear(...), etc.
${#dates.dayOfWeek(date)}		 //arrayDayOfWeek(...), listDayOfWeek(...), etc.
${#dates.dayOfWeekName(date)}//arrayDayOfWeekName(...), listDayOfWeekName(...), etc.
${#dates.dayOfWeekNameShort(date)}//arrayDayOfWeekNameShort(...), listDayOfWeekNameShort(...), etc.
${#dates.hour(date)}			 //arrayHour(...), listHour(...), etc.
${#dates.minute(date)}			 //arrayMinute(...), listMinute(...), etc.
${#dates.second(date)}           //arraySecond(...), listSecond(...), etc.
${#dates.millisecond(date)}      //arrayMillisecond(...), listMillisecond(...), etc.

//创建日期
${#dates.create(year,month,day)}
${#dates.create(year,month,day,hour,minute)}
${#dates.create(year,month,day,hour,minute,second)}
${#dates.create(year,month,day,hour,minute,second,millisecond)
${#dates.createNow()}
${#dates.createNowForTimeZone()}
${#dates.createToday()}			 //time设置为00:00
${#dates.createTodayForTimeZone()}
```

### 日历

**#calendars**：`java.util.Calendar`对象：

```html
//转换为本地日期格式
${#calendars.format(cal)}
${#calendars.arrayFormat(calArray)}
${#calendars.listFormat(calList)}
${#calendars.setFormat(calSet)}

//格式日期采用ISO8601格式
${#calendars.formatISO(cal)}
${#calendars.arrayFormatISO(calArray)}
${#calendars.listFormatISO(calList)}
${#calendars.setFormatISO(calSet)}

//格式日期使用指定的模式
${#calendars.format(cal, 'dd/MMM/yyyy HH:mm')}
${#calendars.arrayFormat(calArray, 'dd/MMM/yyyy HH:mm')}
${#calendars.listFormat(calList, 'dd/MMM/yyyy HH:mm')}
${#calendars.setFormat(calSet, 'dd/MMM/yyyy HH:mm')}

//获取日期属性
${#calendars.day(date)}             //arrayDay(...), listDay(...), etc.
${#calendars.month(date)}           //arrayMonth(...), listMonth(...), etc.
${#calendars.monthName(date)}		//arrayMonthName(...), listMonthName(...), etc.
${#calendars.monthNameShort(date)}  //arrayMonthNameShort(...), listMonthNameShort(...), etc.
${#calendars.year(date)}            //arrayYear(...), listYear(...), etc.
${#calendars.dayOfWeek(date)}       //arrayDayOfWeek(...), listDayOfWeek(...), etc.
${#calendars.dayOfWeekName(date)}   //arrayDayOfWeekName(...), listDayOfWeekName(...), etc.
${#calendars.dayOfWeekNameShort(date)} //arrayDayOfWeekNameShort(...), listDayOfWeekNameShort(...), etc.
${#calendars.hour(date)}            //arrayHour(...), listHour(...), etc.
${#calendars.minute(date)}          //arrayMinute(...), listMinute(...), etc.
${#calendars.second(date)}          //arraySecond(...), listSecond(...), etc.
${#calendars.millisecond(date)}  //arrayMillisecond(...), listMillisecond(...), etc.

//创建日期
${#calendars.create(year,month,day)}
${#calendars.create(year,month,day,hour,minute)}
${#calendars.create(year,month,day,hour,minute,second)}
${#calendars.create(year,month,day,hour,minute,second,millisecond)}
${#calendars.createForTimeZone(year,month,day,timeZone)}
${#calendars.createForTimeZone(year,month,day,hour,minute,timeZone)}
${#calendars.createForTimeZone(year,month,day,hour,minute,second,timeZone)}
${#calendars.createForTimeZone(year,month,day,hour,minute,second,millisecond,timeZone)}
${#calendars.createNow()}
${#calendars.createNowForTimeZone()}
${#calendars.createToday()}
${#calendars.createTodayForTimeZone()}
```

### 数字

```html
//设置数值的整数部分允许的最小位数,例如33 =》 033
${#numbers.formatInteger(num,3)}
${#numbers.arrayFormatInteger(numArray,3)}
${#numbers.listFormatInteger(numList,3)}
${#numbers.setFormatInteger(numSet,3)}

//设置数值的整数部分允许的最小位数和千分位分割符
${#numbers.formatInteger(num,3,'POINT')}
${#numbers.arrayFormatInteger(numArray,3,'POINT')}
${#numbers.listFormatInteger(numList,3,'POINT')}
${#numbers.setFormatInteger(numSet,3,'POINT')}

//设置数值的整数部分允许的最小位数和(精确的)小数位数
${#numbers.formatDecimal(num,3,2)}
${#numbers.arrayFormatDecimal(numArray,3,2)}
${#numbers.listFormatDecimal(numList,3,2)}
${#numbers.setFormatDecimal(numSet,3,2)}

//设置数值的整数部分允许的最小位数和(精确的)小数位数以及小数位分割符
${#numbers.formatDecimal(num,3,2,'COMMA')}
${#numbers.arrayFormatDecimal(numArray,3,2,'COMMA')}
${#numbers.listFormatDecimal(numList,3,2,'COMMA')}
${#numbers.setFormatDecimal(numSet,3,2,'COMMA')}

//设置数值的整数部分允许的最小位数和(精确的)小数位数以及千分位、小数位分割符
${#numbers.formatDecimal(num,3,'POINT',2,'COMMA')}
${#numbers.arrayFormatDecimal(numArray,3,'POINT',2,'COMMA')}
${#numbers.listFormatDecimal(numList,3,'POINT',2,'COMMA')}
${#numbers.setFormatDecimal(numSet,3,'POINT',2,'COMMA')}

//货币格式加￥
${#numbers.formatCurrency(num)}
${#numbers.arrayFormatCurrency(numArray)}
${#numbers.listFormatCurrency(numList)}
${#numbers.setFormatCurrency(numSet)}

//设置数值的整数部分允许的最小位数和小数精确位
${#numbers.formatPercent(num, 3, 2)}
${#numbers.arrayFormatPercent(numArray, 3, 2)}
${#numbers.listFormatPercent(numList, 3, 2)}
${#numbers.setFormatPercent(numSet, 3, 2)}

//创建序列从X到Y
${#numbers.sequence(from,to)}
${#numbers.sequence(from,to,step)}
```

### 字符串

```html
//NULL值安全
${#strings.toString(obj)}                           // array*, list* and set*

//检查字符串是否为空(或空)。检查前执行trim()操作
${#strings.isEmpty(name)}
${#strings.arrayIsEmpty(nameArr)}
${#strings.listIsEmpty(nameList)}
${#strings.setIsEmpty(nameSet)}

//对字符串执行'isEmpty()'检查，如果为false，则返回。如果为真，返回另一个指定的字符串
${#strings.defaultString(text,default)}
${#strings.arrayDefaultString(textArr,default)}
${#strings.listDefaultString(textList,default)}
${#strings.setDefaultString(textSet,default)}

//检查字符串是否包含指定子串
${#strings.contains(name,'ez')}                     // array*, list* and set*
${#strings.containsIgnoreCase(name,'ez')}           // array*, list* and set*

//检查字符串是否以此开头或者结尾
${#strings.startsWith(name,'Don')}                  // array*, list* and set*
${#strings.endsWith(name,endingFragment)}           // array*, list* and set*

//取出指定范围子串
${#strings.indexOf(name,frag)}                      // array*, list* and set*
${#strings.substring(name,3,5)}                     // array*, list* and set*
${#strings.substringAfter(name,prefix)}             // array*, list* and set*
${#strings.substringBefore(name,suffix)}            // array*, list* and set*
${#strings.replace(name,'las','ler')}               // array*, list* and set*

//追加内容
${#strings.prepend(str,prefix)}                     // also array*, list* and set*
${#strings.append(str,suffix)}                      // also array*, list* and set*

//大小写转换
${#strings.toUpperCase(name)}                       // array*, list* and set*
${#strings.toLowerCase(name)}                       // array*, list* and set*

//Split 和 Join
${#strings.arrayJoin(namesArray,',')}
${#strings.listJoin(namesList,',')}
${#strings.setJoin(namesSet,',')}
${#strings.arraySplit(namesStr,',')}                // returns String[]
${#strings.listSplit(namesStr,',')}                 // returns List<String>
${#strings.setSplit(namesStr,',')}                  // returns Set<String>

//trim操作
${#strings.trim(str)}                               // array*, list* and set*

//获取长度
${#strings.length(str)}                             // array*, list* and set*

//当str的长度小于maxWidth的，则返回str
//当str的长度大于maxWidth的，显示maxWidth位（最后三位用...代替）
//当maxWidth小于4时，抛出IllegalArgumentException异常
${#strings.abbreviate(str,10)}                      // also array*, list* and set*

//将第一个字符转换为大写或者小写
${#strings.capitalize(str)}                         // array*, list* and set*
${#strings.unCapitalize(str)}                       // array*, list* and set*

//将第一个单词转换为大写或者小写
${#strings.capitalizeWords(str)}                    // array*, list* and set*
${#strings.capitalizeWords(str,delimiters)}         // array*, list* and set*

//转义/不转义操作
${#strings.escapeXml(str)}                          // array*, list* and set*
${#strings.escapeJava(str)}                         // array*, list* and set*
${#strings.escapeJavaScript(str)}                   // array*, list* and set*
${#strings.unescapeJava(str)}                       // array*, list* and set*
${#strings.unescapeJavaScript(str)}                 // array*, list* and set*

//NULL安全的比较，连接，替换
${#strings.equals(first, second)}
${#strings.equalsIgnoreCase(first, second)}
${#strings.concat(values...)}
${#strings.concatReplaceNulls(nullValue, values...)}

//生成指定长度的字母和数字的随机组合字符串
${#strings.randomAlphanumeric(count)}
```

### 对象

```html
//如果对象是null返回default
${#objects.nullSafe(obj,default)}
${#objects.arrayNullSafe(objArray,default)}
${#objects.listNullSafe(objList,default)}
${#objects.setNullSafe(objSet,default)}
```

### 布尔

```html
//像th:if一样判断是否为true
${#bools.isTrue(obj)}
${#bools.arrayIsTrue(objArray)}
${#bools.listIsTrue(objList)}
${#bools.setIsTrue(objSet)}

//像th:unless一样判断是否为true
${#bools.isFalse(cond)}
${#bools.arrayIsFalse(condArray)}
${#bools.listIsFalse(condList)}
${#bools.setIsFalse(condSet)}

//判断具有and操作
${#bools.arrayAnd(condArray)}
${#bools.listAnd(condList)}
${#bools.setAnd(condSet)}

//判断就有or操作
${#bools.arrayOr(condArray)}
${#bools.listOr(condList)}
${#bools.setOr(condSet)}
```

### 数组

```html
//转换为数组
${#arrays.toArray(object)}

//转换为指定组件类的数组
${#arrays.toStringArray(object)}
${#arrays.toIntegerArray(object)}
${#arrays.toLongArray(object)}
${#arrays.toDoubleArray(object)}
${#arrays.toFloatArray(object)}
${#arrays.toBooleanArray(object)}

//返回数组长度
${#arrays.length(array)}

//判断数组是否为空
${#arrays.isEmpty(array)}

//判断数组是否包含此元素
${#arrays.contains(array, element)}
${#arrays.containsAll(array, elements)}
```

### list

```html
//转换为list
${#lists.toList(object)}

//返回list大小
${#lists.size(list)}

//检查list是否为空
${#lists.isEmpty(list)}

//判断list中是否含有指定元素
${#lists.contains(list, element)}
${#lists.containsAll(list, elements)}

//排序list
${#lists.sort(list)}
${#lists.sort(list, comparator)}
```

### set

```html
//转换为set
${#sets.toSet(object)}

//返回set大小
${#sets.size(set)}

//检查set是否为空
${#sets.isEmpty(set)}

//判断set中是否含有指定元素
${#sets.contains(set, element)}
${#sets.containsAll(set, elements)}
```

### Map

```html
//返回map大小
${#maps.size(map)}

//检查map是否为空
${#maps.isEmpty(map)}

//判断map中是否含有指定元素
${#maps.containsKey(map, key)}
${#maps.containsAllKeys(map, keys)}
${#maps.containsValue(map, value)}
${#maps.containsAllValues(map, value)}
```

### Aggregates

```html
//对集合求和
${#aggregates.sum(array)}
${#aggregates.sum(collection)}

//对集合求平局值
${#aggregates.avg(array)}
${#aggregates.avg(collection)}
```

### IDs

用于处理`id`可能重复的方法

```html
//通常用于th:id属性，用于向id属性值追加计数器
${#ids.seq('someId')}
${#ids.next('someId')}
${#ids.prev('someId')}
```
