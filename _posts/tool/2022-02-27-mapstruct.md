---
layout: post
title: "MapStruct的使用"
date: 2022-02-27 18:16:37
categories: Tool
keywords: "MapStruct,MapStruct的使用"
description: "MapStruct的使用"
---

## 一、简介

​	在一个成熟的工程中，尤其是现在的分布式系统中，应用与应用之间，还有单独的应用细分模块之后，对象需要经过转换包装才能对外提供服务（比如使用VO返回与HTTP相关的出入参，DTO提供与RPC服务相关的出入参）。而对象之间的相互转化成了一个必不可少的工作，这使就需要有一个专门用来解决转换问题的工具，毕竟每一个字段都"Get/Set"会很麻烦。MapStruct就提供了专业的对象之间的转化方式。

### JAVA中数据传输对象的分类

PO（Persistant Object）

用于表示数据库中的一条记录映射成的 java 对象。PO仅仅用于表示数据，没有任何数据操作。通常遵守Java Bean的规范。

---

VO（Value Object）

主要体现在视图的对象，对于一个WEB页面将整个页面的属性封装成一个对象。然后用一个VO对象在控制层与视图层进行传输交换。

---

DTO（Data Transfer Object）

用于表示一个数据传输对象。DTO通常用于不同服务或服务不同分层之间的数据传输。DTO与VO概念相似，并且通常情况下字段也基本一致。但DTO与VO又有一些不同，这个不同主要是设计理念上的，比如API服务需要使用的DTO就可能与VO存在差异。

---

BO（Business Object）

用于表示一个业务对象。BO 包括了业务逻辑，常常封装了对DAO、RPC等的调用，可以进行PO与VO/DTO之间的转换。BO通常位于业务层，在设计上属于被服务层业务流程调用的对象，一个业务流程可能需要调用多个BO来完成。

![对象转换](/img/mapstruct/对象转换.png)

## 二、使用示例

前置条件：引入依赖

```xml
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct</artifactId>
  <version>1.4.2.Final</version>
  <version>[version]</version>
</dependency>
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct-processor</artifactId>
  <version>[version]</version>
  <scope>compile</scope>
</dependency>
```

准备演示基本类

```java
//=====================================数据库对象=====================================
/**
 * 用户信息entity：数据库对应的映射对象
 */
/**
 * 用户信息entity：数据库对应的映射对象
 */
@Data
public class UserInfo {

    /**
     * 用户ID
     */
    private Long id;

    /**
     * 用户名称
     */
    private String name;

    /**
     * 用户出生日期
     */
    private Date birthDate;

    /**
     * @
     */
    private Integer sex;

    /**
     * 账户余额
     */
    private BigDecimal price;
}

/**
 * 用户地址信息entity：数据库对应的映射对象
 */
@Data
public class UserAddressInfo {

    /**
     * 地址ID
     */
    private Long id;

    /**
     * 用户ID
     */
    private Long uid;

    /**
     * 省ID
     */
    private Long provinceId;

    /**
     * 省名
     */
    private String provinceName;

    /**
     * 市ID
     */
    private Long cityId;

    /**
     * 市名
     */
    private String cityName;

    /**
     * 区ID
     */
    private Long countId;

    /**
     * 区名
     */
    private String countyName;
}


//=====================================数据库操作=====================================
/**
 * 模拟数据库操作
 */
@Component
public class UserInfoDAO {

    private static SimpleDateFormat DATA_FORMAT =  new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    @SneakyThrows
    public UserInfo getUser(Long uid) {
        UserInfo userInfo = new UserInfo();
        userInfo.setId(1L);
        userInfo.setName("jack");
        userInfo.setSex(1);
        userInfo.setPrice(new BigDecimal("100.123"));
        userInfo.setBirthDate(DATA_FORMAT.parse("1999-09-19 19:19:19"));
        return userInfo;
    }

    @SneakyThrows
    public List<UserAddressInfo> getAddressInfo(Long uid) {
        UserAddressInfo addressInfo = new UserAddressInfo();
        addressInfo.setId(100L);
        addressInfo.setUid(1L);
        addressInfo.setProvinceId(1L);
        addressInfo.setProvinceName("北京");
        addressInfo.setCityId(1001L);
        addressInfo.setCityName("北京市");
        addressInfo.setCountId(1001003L);
        addressInfo.setCountyName("海淀区");
        return Collections.singletonList(addressInfo);
    }

}

//=====================================返回前端VO=====================================
/**
 * 用户基本信息VO
 */
@Data
@NoArgsConstructor
public class BasicUserInfoVO {

    /**
     * 用户ID
     */
    private Long userId;

    /**
     * 用户名称
     */
    private String name;

    /**
     * 用户出生日期
     */
    private String birthDate;

    /**
     * 性别
     */
    private Integer sex;

    /**
     * 余额
     */
    private String price;

    /**
     * 是否需要账户不足提醒
     */
    private Boolean underAccountReminder;

    public BasicUserInfoVO(BasicUserInfoVO basicUserInfoVO) {
        this.userId = basicUserInfoVO.userId;
        this.name = basicUserInfoVO.name;
        this.birthDate = basicUserInfoVO.birthDate;
        this.sex = basicUserInfoVO.sex;
        this.price = basicUserInfoVO.price;
        this.underAccountReminder = basicUserInfoVO.underAccountReminder;
    }
}


/**
 * 地址信息VO
 */
@Data
public class UserAddressInfoVO {

    /**
     * 地址ID
     */
    private Long addressId;

    /**
     * 用户ID
     */
    private Long uid;

    /**
     * 省ID
     */
    private Long provinceId;

    /**
     * 省名
     */
    private String provinceName;

    /**
     * 市ID
     */
    private Long cityId;

    /**
     * 市名
     */
    private String cityName;

    /**
     * 区ID
     */
    private Long countId;

    /**
     * 区名
     */
    private String countyName;
}

/**
 * 用户全量信息VO
 */
@Data
@NoArgsConstructor
public class UserInfoVO extends BasicUserInfoVO{

    public UserInfoVO(BasicUserInfoVO basicUserInfoVO, List<UserAddressInfoVO> userAddressInfo) {
        super(basicUserInfoVO);
        this.userAddressInfo = userAddressInfo;
    }

    /**
     * 用户地址信息
     */
    private List<UserAddressInfoVO> userAddressInfo;
}
```

### 传统方式

```java
/**
  * 传统方式：手动设置
  */
@Test
public void traditionalWayTest() {
  UserInfo user = userInfoDao.getUser(1L);
  BasicUserInfoVO basicUserInfoVO = new BasicUserInfoVO();
  basicUserInfoVO.setName(user.getName());
  basicUserInfoVO.setUserId(user.getId());
  basicUserInfoVO.setBirthDate(DATA_FORMAT.format(user.getBirthDate()));
  log.info("userInfo={}", JSON.toJSONString(user));
  log.info("basicUserInfoVo={}", JSON.toJSONString(basicUserInfoVO));
}
```

传统方式手动设置每个对象的属性，在属性很多时会耗费太多无用的精力。仔细用心的写，输出结果是正确的

### BeanUtils方式

```java
/**
  * BeanUtils方式
  */
@Test
public void beanUtilsWayTest() {
  UserInfo user = userInfoDao.getUser(1L);
  BasicUserInfoVO basicUserInfoVO = new BasicUserInfoVO();
  BeanUtils.copyProperties(user, basicUserInfoVO);
  log.info("userInfo={}", JSON.toJSONString(user));
  log.info("basicUserInfoVo={}", JSON.toJSONString(basicUserInfoVO));
}
```

输出结果

```log
userInfo={"birthDate":937739959000,"id":1,"name":"jack"} 
basicUserInfoVo={"name":"jack"}
```

BeanUtils当属性名称不一致时，结果是有问题的。且如果转换的是不同包的对象，即使属性名一样，也是无法转换的

### 使用MapStruct

```java
@Mapper
public abstract class UserConvert {

  public static UserConvert INSTANCE = Mappers.getMapper(UserConvert.class);

  @Mappings(
    value = {
      @Mapping(source = "id", target = "userId"),
      @Mapping(source = "name", target = "name"),
      @Mapping(source = "birthDate", target = "birthDate", dateFormat = "yyyy-MM-dd HH:mm:ss")
    }
  )
  public abstract BasicUserInfoVO entity2BasicUserInfoVO(UserInfo userInfo);
}


@Test
public void mapStructTest() {
  UserInfo user = userInfoDao.getUser(1L);
  BasicUserInfoVO basicUserInfoVO = UserConvert.INSTANCE.entity2BasicUserInfoVO(user);
  log.info("userInfo={}", JSON.toJSONString(user));
  log.info("basicUserInfoVo={}", JSON.toJSONString(basicUserInfoVO));
}
```

输出结果

```log
userInfo={"birthDate":937739959000,"id":1,"name":"jack"}
basicUserInfoVo={"birthDate":"1999-09-19 19:19:19","name":"jack","userId":1}
```

MapStruct转换结果完全正确，符合预期

## 三、MapStruct的具体用法

### 类型转换分类

**自动转换**

以下的类型之间是mapstruct自动进行类型转换的

- 基本类型及其他们对应的包装类型，此时MapStruct会自动进行拆装箱，不需要人为的处理
- 基本类型的包装类型和String类型之间

例如：Integer -> int / int -> Integer / int -> String / Integer -> String

**格式化类型转换**

@Mapping(source = "birthDate", target = "birthDate", dateFormat = "yyyy-MM-dd HH:mm:ss")

@Mapping(source = "price", target = "price", numberFormat= #.00")

**自定义类型转换**

自定义属性的转换方式，很多时候需要自定义属性转换能力

@Mapping(target = "price", expression = "java(com.kun.utils.NumberUtils.toNumberRoundUp(userInfo.getPrice(), 2, \"#0.00\"))")

```java
public class NumberUtils {

    public static String toNumberRoundUp(BigDecimal bigDecimal, int newScale, String format) {
        BigDecimal resultDecimal = bigDecimal.setScale(newScale, BigDecimal.ROUND_UP);
        DecimalFormat df = new DecimalFormat(format);
        return df.format(resultDecimal);
    }
}
```

### 常量&默认值&忽略

**设置常类量**

@Mapping(target = "name", constant = "匿名用户")

设置属性值为常量，不需要映射

**忽略**

@Mapping(target = "price", ignore = true)

忽略属性，不设置

**设置默认值**

@Mapping(source = "name", target = "name", defaultValue = "匿名用户")

如果值不存在，使用默认值

### 多对象转一对象

以下代码仅仅是演示，如何从两个对象的参数标识取哪些属性进行组合

```java
@Mappings(
  value = {
    @Mapping(source = "userInfo1.id", target = "userId"),
    @Mapping(source = "userInfo2.name", target = "name", constant = "匿名用户"),
  }
)
public abstract BasicUserInfoVO entity2BasicUserInfoVO(UserInfo userInfo1, UserInfo userInfo2);
```

### 转换之后/之前自定义操作

进行基本的转换之后，有些属性可能需要进行一些自定义操作才能设置正确值

```java
@AfterMapping
public void underAccountReminderJudge(UserInfo userInfo, @MappingTarget BasicUserInfoVO basicUserInfoVO) {
  if (userInfo.getPrice() != null && userInfo.getPrice().compareTo(new BigDecimal("500.00")) >= 0 ) {
    basicUserInfoVO.setUnderAccountReminder(false);
  } else {
    basicUserInfoVO.setUnderAccountReminder(true);
  }
}
```

`@BeforeMapping`和`@AfterMapping`对立，为在映射之前操作

### List转换

将入参是List的参数批量转换为出参是List的出餐

- 当转换类里面有且只有一个入参和出参均和批量参数类型一致的时候会自动循环调用单个出参的方法进行映射

```java
@Mappings(
  value = {
    @Mapping(source = "id", target = "userId"),
    @Mapping(source = "name", target = "name"),
    @Mapping(source = "birthDate", target = "birthDate", dateFormat = "yyyy-MM-dd HH:mm:ss"),
    @Mapping(target = "price", 
             expression = "java(com.kun.utils.NumberUtils.toRoundUp(userInfo.getPrice(), 2, \"#0.00\"))")
  }
)
public abstract BasicUserInfoVO entity2BasicUserInfoVO(UserInfo userInfo);

/**
  * 会自动定位到上面的方法进行循环调用
  */
public abstract List<BasicUserInfoVO> entity2BasicUserInfoVOs(List<UserInfo> userInfo);
```

- 当转换类里面有多个入参和出参均和批量参数类型一致的时候，之间写批量方法会有二义性异常，此项明确指定调用方法名即可

```java
@Mappings(
  value = {
    @Mapping(source = "id", target = "userId"),
    @Mapping(source = "name", target = "name"),
    @Mapping(source = "birthDate", target = "birthDate", dateFormat = "yyyy-MM-dd HH:mm:ss"),
    @Mapping(target = "price", 
             expression = "java(com.kun.utils.NumberUtils.toRoundUp(userInfo.getPrice(), 2, \"#0.00\"))")
  }
)
@Named("entity2BasicUserInfoVO")  // 指定名称
public abstract BasicUserInfoVO entity2BasicUserInfoVO(UserInfo userInfo);

/**
  * 会自动定位到上面的方法进行循环调用
  */
@IterableMapping(qualifiedByName = "entity2BasicUserInfoVO")
public abstract List<BasicUserInfoVO> entity2BasicUserInfoVOs(List<UserInfo> userInfo);
```

### 深拷贝

`@Mapping`、`@Mapper`注解下，都有一个`mappingControl`属性，里面有个`DeepClone.class`是深拷贝

### 多对象带List转一对象

很多时候，对象带有别的类的引用，此时我们可以自己书写代码，组合一下使用

```java
public UserInfoVO entity2UserInfoVO(UserInfo userInfo, List<UserAddressInfo> userAddressInfo) {
  // 先转换userInfo
  BasicUserInfoVO basicUserInfoVO = INSTANCE.entity2BasicUserInfoVO(userInfo);
  // 再转换userAddressInfo
  List<UserAddressInfoVO> userAddressInfoVOs = INSTANCE.entity2UserAddressInfoVOs(userAddressInfo);
  // 塞入返回结果
  return new UserInfoVO(basicUserInfoVO, userAddressInfoVOs);
}
```

### 继承

继承已有的映射规则，减少冗余代码

```java
@Mappings(
  value = {
    @Mapping(source = "id", target = "userId"),
    @Mapping(source = "name", target = "name"),
    @Mapping(source = "birthDate", target = "birthDate", dateFormat = "yyyy-MM-dd HH:mm:ss"),
    @Mapping(target = "price",
             expression = "java(com.kun.utils.NumberUtils.toRoundUp(userInfo.getPrice(), 2, \"#0.00\"))")
  }
)
@Named("entity2BasicUserInfoVO")
public abstract BasicUserInfoVO entity2BasicUserInfoVO(UserInfo userInfo);

/**
  * 更新basicUserInfoVO的属性
  */
@InheritConfiguration(name = "entity2BasicUserInfoVO")
public abstract BasicUserInfoVO updateBasicUserInfoVO(UserInfo userInfo, @MappingTarget BasicUserInfoVO basicUserInfoVO);
```

### 集成Spring

只需要处理以下`@Mapper`注解，就可以用`@Autowire`进行注入

```java
@Mapper(componentModel="spring")
public abstract class UserConvert {
  // 可以删除了
  // public static UserConvert INSTANCE = Mappers.getMapper(UserConvert.class);

  //.......
}
```







