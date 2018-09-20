---
layout: post
title: "thymeleaf与Spring整合"
date: 2018-09-06 15:37:40
categories: thymeleaf
---

# 前言

Thymeleaf集成了Spring Framework的3.x和4.x版本，由两个名为`thymeleaf-spring3`和`thymeleaf-spring4`的独立库提供

# 一、将Thymeleaf与Spring结合起来

Thymeleaf提供了Spring集成，允许将Spring MVC中JSP的全方面替代

- `@Controller`对象中的映射方法转发到由Thymeleaf管理的模板，就像使用JSP一样
- 在模板中使用**Spring Expression Language**（Spring EL）代替OGNL
- 在模板中创建数据绑定，包括使用属性编辑器，转换服务和验证错误处理
- 显示由Spring（通过常用`MessageSource`对象）的国际化消息
- 使用Spring自己的资源解析机制解析模板

# 二、SpringStandard方言

```txt
StandardDialect
 |
 +- SpringStandardDialect
```

SpringStandard引入了以下特定的功能？？？？

- 使用Spring Expression Language（SpEL）作为变量表达式语言，而不是OGNL。因此，`${...}`和`*{...}`表达式都将由Spring的表达式语言引擎进行解析。

- 使用SpringEL的语法访问Application Context中的任何bean： `${@myBean.doSomething()}`？？？？？
- 于表格处理的新属性：th:field,th:errors和th:errorclass,除此还有一个th:object的新实现，允许它被用于形式命令选择？？？
- 一个新的表达式：`#themes.code(...)`，相当于jsp自定义标签中的spring:theme
- 在spring4.0集成中的一个新的表达式：`#mvc.uri(...)`，相当于jsp自定义标签中的`spring:mvcUrl(...)`

一个bean配置示例

```java
@Bean
public SpringResourceTemplateResolver templateResolver(){
    SpringResourceTemplateResolver templateResolver = new SpringResourceTemplateResolver();
    templateResolver.setApplicationContext(this.applicationContext);
    templateResolver.setPrefix("/WEB-INF/templates/");
    templateResolver.setSuffix(".html");
    templateResolver.setTemplateMode(TemplateMode.HTML);
    templateResolver.setCacheable(true);
    return templateResolver;
}

@Bean
public SpringTemplateEngine templateEngine(){
    SpringTemplateEngine templateEngine = new SpringTemplateEngine();
    templateEngine.setTemplateResolver(templateResolver());
    templateEngine.setEnableSpringELCompiler(true);
    return templateEngine;
}
```

或者XML配置

```xml
<bean id="templateResolver"
      class="org.thymeleaf.spring4.templateresolver.SpringResourceTemplateResolver">
  <property name="prefix" value="/WEB-INF/templates/" />
  <property name="suffix" value=".html" />
  <property name="templateMode" value="HTML" />
  <property name="cacheable" value="true" />
</bean>
    
<bean id="templateEngine"
      class="org.thymeleaf.spring4.SpringTemplateEngine">
  <property name="templateResolver" ref="templateResolver" />
  <property name="enableSpringELCompiler" value="true" />
</bean>
```

# 三、 视图和视图解析器

## Spring MVC中的视图和视图解析器

Spring MVC中视图与视图解析器接口

- `org.springframework.web.servlet.View`
- `org.springframework.web.servlet.ViewResolver`

View是负责渲染实际的HTML，通常由一些模板引擎来负责，如JSP和Thymeleaf

ViewResolvers根据Controller返回的字符串数据，定位到资源位置并应用View合成界面

Spring视图解析器配置示例：

```xml
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
  <property name="viewClass" value="org.springframework.web.servlet.view.JstlView" />
  <property name="prefix" value="/WEB-INF/jsps/" />
  <property name="suffix" value=".jsp" />
  <property name="order" value="2" />
  <property name="viewNames" value="*jsp" />
</bean>
```

- `viewClass`合成View实例的类
- `prefix`并`suffix`视图前缀和后缀，用于定位模版
- `order` 在链中查询ViewResolver的顺序
- `viewNames` ViewResolver解析的视图名称

 ## Thymeleaf中的视图和视图解析器

Thymeleaf为上面提到的两个接口提供了实现：

- `org.thymeleaf.spring4.view.ThymeleafView`
- `org.thymeleaf.spring4.view.ThymeleafViewResolver`

Thymeleaf View Resolver的配置与JSP的配置非常相似：

```java
@Bean
public ThymeleafViewResolver viewResolver(){
    ThymeleafViewResolver viewResolver = new ThymeleafViewResolver();
    viewResolver.setTemplateEngine(templateEngine());
    viewResolver.setOrder(1);
    viewResolver.setViewNames(new String[] {".html"});
    return viewResolver;
}
```

或者XML配置

```xml
<bean class="org.thymeleaf.spring4.view.ThymeleafViewResolver">
  <property name="templateEngine" ref="templateEngine" />
  <property name="order" value="1" />
  <property name="viewNames" value="*.html" />
</bean>
```

解释：`prefix`和`suffix`在`SpringResourceTemplateResolver`指定传入到了`SpringTemplateEngine`所以不用指定

# 三、Thymeleaf配置Spring

```java
 @Bean
public SpringResourceTemplateResolver templateResolver(){
    SpringResourceTemplateResolver templateResolver = new SpringResourceTemplateResolver();
    templateResolver.setApplicationContext(this.applicationContext);
    templateResolver.setPrefix("/WEB-INF/templates/");
    templateResolver.setSuffix(".html");
    templateResolver.setTemplateMode(TemplateMode.HTML);
    templateResolver.setCacheable(true);
    return templateResolver;
}

@Bean
public SpringTemplateEngine templateEngine(){
    SpringTemplateEngine templateEngine = new SpringTemplateEngine();
    templateEngine.setTemplateResolver(templateResolver());
    templateEngine.setEnableSpringELCompiler(true);
    return templateEngine;
}

@Bean
public ThymeleafViewResolver viewResolver(){
    ThymeleafViewResolver viewResolver = new ThymeleafViewResolver();
    viewResolver.setTemplateEngine(templateEngine());
    return viewResolver;
}
```

或者XML配置

```xml
<bean id="templateResolver"
      class="org.thymeleaf.spring4.templateresolver.SpringResourceTemplateResolver">
  <property name="prefix" value="/WEB-INF/templates/" />
  <property name="suffix" value=".html" />
  <property name="templateMode" value="HTML" />
  <property name="cacheable" value="true" />
</bean>
    
<bean id="templateEngine"
      class="org.thymeleaf.spring4.SpringTemplateEngine">
  <property name="templateResolver" ref="templateResolver" />
  <property name="enableSpringELCompiler" value="true" />
</bean>

<bean class="org.thymeleaf.spring4.view.ThymeleafViewResolver">
  <property name="templateEngine" ref="templateEngine" />
  <property name="order" value="1" />
  <property name="viewNames" value="*.html" />
</bean>
```

# 四、使用Spring Framework的配置

## 使用Message解析

```java
@Bean
public ResourceBundleMessageSource messageSource() {
    ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
    messageSource.setBasename("Messages");
    return messageSource;
}
```
会加载类路径下的Message_xx.properties文件作为国际化文件，`#{...}`使用国际化

```html
#{bool.true}
#{date.format}
#{foot.msg}

拼接字符串用法
<td th:text="#{|bool.${sb.covered}|}">yes</td>
```

## 使用转换服务

```java
public class DateFormatter implements Formatter<Date> {

    @Autowired
    private MessageSource messageSource;

    public DateFormatter() {
        super();
    }

    @Override
    public Date parse(final String text, final Locale locale) throws ParseException 	{
        final SimpleDateFormat dateFormat = createDateFormat(locale);
        return dateFormat.parse(text);
    }

    @Override
    public String print(final Date object, final Locale locale) {
        final SimpleDateFormat dateFormat = createDateFormat(locale);
        return dateFormat.format(object);
    }

    private SimpleDateFormat createDateFormat(final Locale locale) {
        //从Spring Message中获取日期格式
        final String format = this.messageSource.getMessage("date.format", null, locale);
        final SimpleDateFormat dateFormat = new SimpleDateFormat(format);
        dateFormat.setLenient(false);
        return dateFormat;
    }

}
```

使用`{{...}}`转换服务

```html
<td th:text="${{sb.datePlanted}}">13/01/2011</td>
```

## th:field标签

```html
<form action="@{/doCommit}" method="POST" th:object="${person}">
  <p>
    <!-- th:field相当于同时设置id、name、value属性 -->
	<label for="personName" th:text="姓名">姓名</label>	
	<input type="text" th:field="*{personName}">
  </p>
  <p>
    <!-- th:field和th:selected不可以同时使用。th:field会生成id、name但无法指定选定的值-->
	<label for="city" th:text="出生城市">城市</label>
	<select id="city" name="city">
	  <option th:each="item:${citys}" 
            th:value="${item.id}" 
            th:text="${item.cityName}" 
            th:selected="${item.id} == *{city.id}"></option>
	</select>
  </p>
  <p>
    <label th:text="想居住城市">想居住城市</label>
	<!-- 使用th:block th:each构造块，
		th:field会生成name和自增的id,th:field和th:checked不可以同时使用-->
	<!--/*/ <th:block th:each="item:${citys}"> /*/-->
	<label th:for="${#ids.next('liveCitys')}" th:text="${item.cityName}"></label>
	<input type="checkbox" th:id="${#ids.seq('liveCitys')}" name="liveCitys"
	th:value="${item.id}" th:checked="${#lists.contains(person.liveCitys,item)}" >
	<!--/*/ </th:block> /*/-->
  </p>
  <input type="submit" value="提交">
</form>
```
## CheckBox字段

```html
<form action="" th:object="${user}">
    <label th:for="${#ids.next('goodMan')}"></label>	<!--需要使用next拿到最后的id-->
    <input type="checkbox" th:field="*{goodMan}" /> 	<!--这个id先生成goodMan1-->
</form>

<!--数组类型的CheckBox-->
<ul>
  <li th:each="feat : ${allFeatures}">
    <input type="checkbox" th:field="*{features}" th:value="${feat}" />
    <!--需要使用prev拿到前面的id-->
    <label th:for="${#ids.prev('features')}" 
           th:text="#{${'seedstarter.feature.' + feat}}">Heating</label>
  </li>
</ul>
```

对应生成HTML

```html
<form action="">
	<label for="goodMan1"></label>
	<input type="checkbox" id="goodMan1" name="goodMan" value="true" />
    <input type="hidden" name="_goodMan" value="on"/>  
</form>

<ul>
  <li>
    <input id="features1" name="features" type="checkbox" value="SEEDSTARTER_SPECIFIC_SUBSTRATE" />
    <input name="_features" type="hidden" value="on" />
    <label for="features1">Seed starter-specific substrate</label>
  </li>
  <li>
    <input id="features2" name="features" type="checkbox" value="FERTILIZER" />
    <input name="_features" type="hidden" value="on" />
    <label for="features2">Fertilizer used</label>
  </li>
  <li>
    <input id="features3" name="features" type="checkbox" value="PH_CORRECTOR" />
    <input name="_features" type="hidden" value="on" />
    <label for="features3">PH Corrector used</label>
  </li>
</ul>
```

## Radio字段

```html
<ul>
    <li th:each="ele : ${hobbys}">
        <input type="radio" th:field="${user.hobby}" th:value="${ele}" />
        <label th:for="${#ids.prev('hobby')}" 
               th:text="#{|user.hobby.${ele}|}">Heating</label>
    </li>
</ul>
```

对应生成HTML

```html
<ul>
    <li>
        <input type="radio" value="AAA" id="hobby1" name="hobby" checked="checked" />
        <label for="hobby1" >篮球</label>
    </li>
    <li>
        <input type="radio" value="BBB" id="hobby2" name="hobby" />
        <label for="hobby2" >羽毛球</label>
    </li>
    <li>
        <input type="radio" value="CCC" id="hobby3" name="hobby" />
        <label for="hobby3" >排球</label>
    </li>
</ul>
```

## 下拉列表

```html
<select	th:field="${user.hobby}">
    <option th:each="ele : ${hobbys}"
            th:text="#{|user.hobby.${ele}|}" 
            th:value="${ele}">
</select>
```
对应生成HTML

```html
<select id="hobby" name="hobby">
    <option value="AAA" >篮球
    <option value="BBB" selected="selected">羽毛球
    <option value="CCC">排球
</select>
```

# 五、校验和错误消息

错误必须使用JSR303规范校验，只能使用javax.validation.constraints.*注解标注，在java代码里需要校验的对象要加@Validated注解

## 字段错误

pom需要引入依赖

```xml
<!--JSR303标注-->
<dependency>
    <groupId>javax.validation</groupId>
    <artifactId>validation-api</artifactId>
    <version>1.1.0.Final</version>
    <scope>compile</scope>
</dependency>

<!--Hibernate对JSR303的实现-->
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>5.3.2.Final</version>
</dependency>
```

java代码写法

```java
@RequestMapping("/add")
public String add(@Validated   User user,BindingResult result) {
    if(result.hasErrors()){
        for (ObjectError error : result.getAllErrors()) {
            System.out.println(error.getDefaultMessage());
        }
        return "seedstartermng";
    }
    return "seedstartermng";
}
```

HTML代码写法

```html
<form action="" th:action="@{/add}" method="post" th:object="${user}">
  <!--遍历所有错误描述，进行打印-->
  <ul>
	<li th:each="err : ${#fields.errors('age')}" th:text="${err}" />
  </ul>
  <label th:for="${#ids.next('age')}">Age</label>
  <!--输入框会被标红-->
  <input type="text" th:field="*{age}" th:class="${#fields.hasErrors('age')}? fieldError" >
    <div>
        <!--显示此错误描述-->
        <span th:if="${#fields.hasErrors('age')}" th:errors="*{age}"></span>
    </div>
    <input type="submit">
</form> 
```

简化CSS标签

```html
<input type="text" th:field="*{age}" th:class="${#fields.hasErrors('age')}? fieldError" >
<!--可以简化为-->
<input type="text" th:field="*{age}" th:errorClass="fieldError" >
```

## 所有错误

`*`，`all`代表所有错误。

`#fields.hasErrors('*')`=`#fields.hasAnyErrors()`=`#fields.errors('*')`=`#fields.allErrors()`

```html
<ul>
  <li th:each="err : ${#fields.errors('*')}" th:text="${err}" />
</ul>
```

## 全局错误

```html
<!-- 第一种遍历方式 -->
<ul th:if="${#fields.hasErrors('global')}">
  <li th:each="err : ${#fields.errors('global')}" th:text="${err}"></li>
</ul>
<!-- 第二种遍历方式 -->
<p th:if="${#fields.hasErrors('global')}" th:errors="*{global}"></p>
<!-- 第三种遍历方式 -->
<div th:if="${#fields.hasGlobalErrors()}">
  <p th:each="err : ${#fields.globalErrors()}" th:text="${err}">...</p>
</div>
```

# 六、转换服务

## 配置转换服务

通过扩展Spring的`WebMvcConfigurerAdapter`适配器，注册转换服务

```java
@Override
public void addFormatters(final FormatterRegistry registry) {
    super.addFormatters(registry);
    registry.addFormatter(dateFormatter());	//实现Formatter<Date>接口
}

@Bean
public DateFormatter dateFormatter() {
    return new DateFormatter();
}
```

```java
public class DateFormatter implements Formatter<Date> {
    @Autowired
    private MessageSource messageSource;

    @Override
    public Date parse(final String text, final Locale locale) throws ParseException {
        final SimpleDateFormat dateFormat = createDateFormat(locale);
        return dateFormat.parse(text);
    }

    @Override
    public String print(final Date object, final Locale locale) {
        final SimpleDateFormat dateFormat = createDateFormat(locale);
        return dateFormat.format(object);
    }

    private SimpleDateFormat createDateFormat(final Locale locale) {
        final String format = this.messageSource.getMessage("date.format", null, locale);
        final SimpleDateFormat dateFormat = new SimpleDateFormat(format);
        dateFormat.setLenient(false);
        return dateFormat;
    }

}
```

## 使用转换服务

- 对于变量表达式： `${{...}}`
- 对于选择表达式： `*{{...}}`

## #conversions对象

手动调用转换服务，`#conversions.convert(Object,ClassName)`：将对象转换为指定的类，例如${#conversions.convert(val,'String')}



