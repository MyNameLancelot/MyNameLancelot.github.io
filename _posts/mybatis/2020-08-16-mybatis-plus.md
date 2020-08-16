---
layout: post
title: "mybatis plus使用指南"
date: 2020-08-16 20:48:02
categories: mybatis
keywords: "mybatis,mybatis plus"
description: "mybatis plus使用指南"
---

## 一、集成Spring boot

```xml
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>mybatis-plus-boot-starter</artifactId>
  <version>[version]</version>
</dependency>

<!-- 用于生成逆向工程 -->
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>mybatis-plus-generator</artifactId>
  <version>[version]</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.freemarker</groupId>
  <artifactId>freemarker</artifactId>
  <version>[version]</version>
  <scope>test</scope>
</dependency>
```

## 二、逆向工程配置

```java
public class CodeGenerator {

  private static final String PROJECT_PATH = System.getProperty("user.dir");
  // 表名
  private static final String TABLE_NAME = "user";
  // 逻辑删除字段
  private static final String LOGIC_DELETE_FIELD_NAME = "is_delete";
  // 是否覆盖
  private static final Boolean FILE_OVERRIDE = true;
  // 模块名称
  private static final String MODULE_NAME = "user";
  
  // 数据库配置
  private static final String DATABASE_URL = "jdbc:mysql://localhost:3306/book?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai";
  private static final String DATABASE_DRIVER = "com.mysql.cj.jdbc.Driver";
  private static final String DATABASE_USER_NAME = "root";
  private static final String DATABASE_PASSWORD = "123456";

  @Test
  public void generator() {
    // 代码生成器
    AutoGenerator autoGenerator = new AutoGenerator();

    // 全局配置
    autoGenerator.setGlobalConfig(getGlobalConfig());

    // 数据源配置
    autoGenerator.setDataSource(getDataSourceConfig());

    // 包配置
    autoGenerator.setPackageInfo(getPackageConfig());

    // 自定义配置
    autoGenerator.setCfg(getInjectionConfig());

    // 如需配置自定义输出模板，可以对以下对象进行扩展
    TemplateConfig templateConfig = new TemplateConfig();
    // 自定义路径，不需要默认的xml路径再次生成
    templateConfig.setXml(null);
    autoGenerator.setTemplate(templateConfig);

    // 策略配置，除基本设置以为还具有以下功能：
    // 可以配置各层的父类，和实体类公用的字段【公用字段在子类不在生成】
    // 可以设置表前缀
    // 可以设置Controller的请求路径驼峰转连字符
    autoGenerator.setStrategy(getStrategyConfig());

    // 设置模板引擎
    autoGenerator.setTemplateEngine(new FreemarkerTemplateEngine());

    autoGenerator.execute();
  }

  private InjectionConfig getInjectionConfig() {

    // 如要自定义模板，可添加模板里面的变量
    InjectionConfig injectionConfig = new InjectionConfig() {
      @Override
      public void initMap() {
      }
    };
    // 如果模板引擎是 freemarker
    String templatePath = "/templates/mapper.xml.ftl";
    // 自定义输出配置
    List<FileOutConfig> fileOutConfigs = new ArrayList<>();
    // 自定义配置会被优先输出，默认xml生成在对应mapper下，一般需要更改
    fileOutConfigs.add(new FileOutConfig(templatePath) {
      @Override
      public String outputFile(TableInfo tableInfo) {
        // 自定义输出文件名 ， 如果你 Entity 设置了前后缀、此处注意 xml 的名称会跟着发生变化！！
        return PROJECT_PATH + "/src/main/resources/mapper/" +
          (StringUtils.isBlank(MODULE_NAME) ? "" : MODULE_NAME + "/")
          + tableInfo.getEntityName() + "Mapper" + StringPool.DOT_XML;
      }
    });
    injectionConfig.setFileOutConfigList(fileOutConfigs);
    return injectionConfig;
  }

  private StrategyConfig getStrategyConfig() {
    StrategyConfig strategy = new StrategyConfig();
    // 字段与属性命名策略
    strategy.setNaming(NamingStrategy.underline_to_camel);
    strategy.setColumnNaming(NamingStrategy.underline_to_camel);
    // 是否使用Lombok插件
    strategy.setEntityLombokModel(true);
    // 是否使用RestController注解
    strategy.setRestControllerStyle(true);
    // 实体类是否实现JDK序列化接口
    strategy.setEntitySerialVersionUID(false);
    // 实体类是否标上字段注解
    strategy.setEntityTableFieldAnnotationEnable(true);
    // 设置逻辑删除字段
    if(StringUtils.isNotBlank(LOGIC_DELETE_FIELD_NAME)) {
      strategy.setLogicDeleteFieldName(LOGIC_DELETE_FIELD_NAME);
    }
    strategy.setInclude(TABLE_NAME);
    return strategy;
  }

  private PackageConfig getPackageConfig() {
    PackageConfig pc = new PackageConfig();
    pc.setModuleName(MODULE_NAME);
    // 设置各个层包名
    pc.setParent("com.kun.mybatisplus");
    pc.setController("controller");
    pc.setEntity("domain");
    pc.setMapper("mapper");
    pc.setService("service");
    return pc;
  }

  private DataSourceConfig getDataSourceConfig() {
    DataSourceConfig dataSourceConfig = new DataSourceConfig();
    dataSourceConfig.setUrl(DATABASE_URL);
    dataSourceConfig.setDriverName(DATABASE_DRIVER);
    dataSourceConfig.setUsername(DATABASE_USER_NAME);
    dataSourceConfig.setPassword(DATABASE_PASSWORD);
    // 各种数据库配置
    dataSourceConfig.setDbType(DbType.MYSQL);
    dataSourceConfig.setKeyWordsHandler(new MySqlKeyWordsHandler());
    dataSourceConfig.setTypeConvert(new MySqlTypeConvert());
    return dataSourceConfig;
  }

  private GlobalConfig getGlobalConfig() {
    GlobalConfig globalConfig = new GlobalConfig();
    globalConfig.setOutputDir(PROJECT_PATH + "/src/main/java");
    globalConfig.setAuthor("WangYuKun");
    globalConfig.setOpen(false);
    // 实体属性 Swagger2 注解
    globalConfig.setSwagger2(true);
    // 生成的service名称，默认I%sService
    globalConfig.setServiceName("%sService");
    // 是否覆盖
    globalConfig.setFileOverride(FILE_OVERRIDE);
    // XML是否生成基本列
    globalConfig.setBaseColumnList(true);
    // XML是否生成基本映射
    globalConfig.setBaseResultMap(true);
    // 是否启用缓存
    globalConfig.setEnableCache(false);
    // 主键策略
    globalConfig.setIdType(IdType.AUTO);
    return globalConfig;
  }
}
```

## 三、基础Mapper的使用

### 插入方法的使用

```java
public void testInsert() {
  User user = new User();
  user.setName("Rose");
  user.setAge(18);
  user.setSex(SexEnum.FEMALE);
  user.setEmail("123@qq.com");
  // 只有存在的值属性对应得字段才会插入
  userMapper.insert(user);
}
```

> 设置性别使用了枚举的形式，请查看<a href="#enum">枚举用法</a>

### 更新方法的使用

```java
public void testUpdate() {
  User user = new User();
  user.setId(6L);
  user.setName("Jmi");
  user.setVersion(1);
  userMapper.updateById(user);
  userMapper.update(user, Wrappers.<User>lambdaQuery().eq(User::getName, "Jack"));
}
```

> 更新使用了乐观锁机制，请查看<a href="#optimisticLocker">乐观锁插件</a>

### 删除方法的使用

```java
public void testDelete() {
  userMapper.deleteById(6L);
  userMapper.delete(Wrappers.<User>lambdaQuery().eq(User::getName, "Jmi"));
}
```

> 删除使用逻辑删除，请查看<a href="#logicDelete">逻辑删除设置</a>

### 查询方法的使用

```java
public void testSelect() {
  User userById = userMapper.selectById(1);
  log.info("用户{}", userById);

  // selectBatchIds
  List<User> usersListByIds = userMapper.selectBatchIds(Arrays.asList("1", "2"));
  log.info("用户列表{}", usersListByIds);

  // QueryWrapper与selectOne组合使用
  QueryWrapper<User> queryWrapper = new QueryWrapper<>();
  queryWrapper.select(User.class, (col) -> !col.isVersion());
  queryWrapper.eq("id", 1);
  User userByWrapper = userMapper.selectOne(queryWrapper);
  log.info("用户{}", userByWrapper);

  // LambdaQueryWrapper与selectList组合使用
  LambdaQueryWrapper<User> userLambdaQueryWrapper = new LambdaQueryWrapper<>();
  userLambdaQueryWrapper.gt(true, User::getAge, 18);
  List<User> usersByLambdaWrapper = userMapper.selectList(userLambdaQueryWrapper);
  log.info("用户列表{}", usersByLambdaWrapper);

  // selectByMap,注意必须使用数据库列名
  Map<String, Object> params = new HashMap<>();
  params.put("name", "Jone");
  List<User> usersByMap = userMapper.selectByMap(params);
  log.info("用户列表{}", usersByMap);

  // selectCount
  Integer count = userMapper.selectCount(Wrappers.<User>lambdaQuery().gt(User::getAge, 20));
  log.info("20岁以上总计{}人", count);

  // 分页
  Page<User> page = new Page<>(2, 2);
  // 排序字段放入Wrapper是不会返回排序字段的，放入Page对象会使用它作为查询排序条件饼返回排序字段
  page.addOrder(OrderItem.asc("id"));
  Page<User> userPage = userMapper.selectPage(page, null);
  log.info("userPage信息 current={}", userPage.getCurrent());
  log.info("userPage信息 size={}", userPage.getSize());
  log.info("userPage信息 pages={}", userPage.getPages());
  log.info("userPage信息 total={}", userPage.getTotal());
  log.info("userPage信息 orders={}", userPage.getOrders());
  log.info("userPage信息 records={}", userPage.getRecords());
}
```

> 如果不使用分页插件则分页为逻辑分页，可使用<a href="#page">分页插件达到物理分页</a>

## 四、基础Service的使用

### 插入方法的使用

```java
// 插入一条记录（选择字段，策略插入）
boolean save(T entity);
// 插入（批量）
boolean saveBatch(Collection<T> entityList);
// 插入（批量）
boolean saveBatch(Collection<T> entityList, int batchSize);
```

### 更新方法的使用

```java
// 根据 UpdateWrapper 条件，更新记录 需要设置sqlset
boolean update(Wrapper<T> updateWrapper);
// 根据 whereEntity 条件，更新记录
boolean update(T entity, Wrapper<T> updateWrapper);
// 根据 ID 选择修改
boolean updateById(T entity);
// 根据ID 批量更新
boolean updateBatchById(Collection<T> entityList);
// 根据ID 批量更新
boolean updateBatchById(Collection<T> entityList, int batchSize);
```

### 插入&更新的使用

```java
// TableId 注解存在更新记录，否插入一条记录
boolean saveOrUpdate(T entity);
// 根据updateWrapper尝试更新，否继续执行saveOrUpdate(T)方法
boolean saveOrUpdate(T entity, Wrapper<T> updateWrapper);
// 批量修改插入
boolean saveOrUpdateBatch(Collection<T> entityList);
// 批量修改插入
boolean saveOrUpdateBatch(Collection<T> entityList, int batchSize);
```

### 删除方法的使用

```java
// 根据 entity 条件，删除记录
boolean remove(Wrapper<T> queryWrapper);
// 根据 ID 删除
boolean removeById(Serializable id);
// 根据 columnMap 条件，删除记录
boolean removeByMap(Map<String, Object> columnMap);
// 删除（根据ID 批量删除）
boolean removeByIds(Collection<? extends Serializable> idList);
```

### 查询单个方法的使用

```java
// 根据 ID 查询
T getById(Serializable id);
// 根据 Wrapper，查询一条记录。结果集，如果是多个会抛出异常，随机取一条加上限制条件 wrapper.last("LIMIT 1")
T getOne(Wrapper<T> queryWrapper);
// 根据 Wrapper，查询一条记录
T getOne(Wrapper<T> queryWrapper, boolean throwEx);
// 根据 Wrapper，查询一条记录
Map<String, Object> getMap(Wrapper<T> queryWrapper);
// 根据 Wrapper，查询一条记录
<V> V getObj(Wrapper<T> queryWrapper, Function<? super Object, V> mapper);
```

### 查询列表方法的使用

```java
// 查询所有
List<T> list();
// 查询列表
List<T> list(Wrapper<T> queryWrapper);
// 查询（根据ID 批量查询）
Collection<T> listByIds(Collection<? extends Serializable> idList);
// 查询（根据 columnMap 条件）
Collection<T> listByMap(Map<String, Object> columnMap);
// 查询所有列表
List<Map<String, Object>> listMaps();
// 查询列表
List<Map<String, Object>> listMaps(Wrapper<T> queryWrapper);
// 查询全部记录
List<Object> listObjs();
// 查询全部记录
<V> List<V> listObjs(Function<? super Object, V> mapper);
// 根据 Wrapper 条件，查询全部记录
List<Object> listObjs(Wrapper<T> queryWrapper);
// 根据 Wrapper 条件，查询全部记录
<V> List<V> listObjs(Wrapper<T> queryWrapper, Function<? super Object, V> mapper);
```

### 分页查询方法的使用

```java
// 无条件分页查询
IPage<T> page(IPage<T> page);
// 条件分页查询
IPage<T> page(IPage<T> page, Wrapper<T> queryWrapper);
// 无条件分页查询
IPage<Map<String, Object>> pageMaps(IPage<T> page);
// 条件分页查询
IPage<Map<String, Object>> pageMaps(IPage<T> page, Wrapper<T> queryWrapper);
```

## 五、插件的使用

### <span id="enum">枚举用法</span>

**第一步：实现枚举类**

```java
//================================== 方式一 ==================================
public enum SexEnum {
  MALE("M", "男"),  FEMALE("F", "女");

  SexEnum(String code, String desc) {
    this.code = code;
    this.desc = desc;
  }

  //标记数据库存的值是code，支持Integer、String
  @EnumValue
  private final String code;

  private final String desc;
}

//================================== 方式二 ==================================
public enum SexEnum implements IEnum<String> {
  MALE("M", "男"),  FEMALE("F", "女");

  private final String code;
  private final String desc;

  SexEnum(String code, String desc) {
    this.code = code;
    this.desc = desc;
  }

  // 返回数据库要存储值
  @Override
  public String getValue() {
    return this.code;
  }
}
```

**第二步：配置要扫描的枚举包**

```properties
mybatis-plus.type-enums-package=com.kun.mybatisplus.domain.enums
```

### <span id="optimisticLocker">乐观锁</span>

**第一步：配置乐观锁插件**

```java
@Bean
public OptimisticLockerInterceptor optimisticLockerInterceptor() {
  return new OptimisticLockerInterceptor();
}
```

**第二步：使用@version对实体类标注乐观锁字段**

```java
public class User {
  
  // ......
  
  @TableField("version")
  @Version
  private Integer version;
}
```

> 乐观锁仅支持 `updateById(id)` 与 `update(entity, wrapper)` 方法

### <span id="logicDelete">逻辑删除</span>

<span style="color:red">在逆向工程中配置逻辑表删除字段，可以自动生成注解。</span>

```java
public class User {

  // ......

  @TableField("is_delete")
  @TableLogic
  private Boolean isDelete;
}

```

### <span id="#page">物理分页</span>

**第一步：物理分页插件设置**

```java
@Bean
public PaginationInterceptor paginationInterceptor() {
  PaginationInterceptor paginationInterceptor = new PaginationInterceptor();
  // 设置请求的页面大于最大页后操作， true调回到首页，false 继续请求  默认false
  // paginationInterceptor.setOverflow(false);
  // 设置最大单页限制数量，默认 500 条，-1 不受限制
  // paginationInterceptor.setLimit(500);
  // 开启 count 的 join 优化,只针对部分 left join
  paginationInterceptor.setCountSqlParser(new JsqlParserCountOptimize(true));
  return paginationInterceptor;
}
```

**第二步：使用分页查询**

```java
public void testSelect() {
  // 分页
  Page<User> page = new Page<>(2, 2);
  // 排序字段放入Wrapper是不会返回排序字段的，放入Page对象会使用它作为查询排序条件饼返回排序字段
  page.addOrder(OrderItem.asc("id"));
  Page<User> userPage = userMapper.selectPage(page, null);
  log.info("userPage信息 current={}", userPage.getCurrent());
  log.info("userPage信息 size={}", userPage.getSize());
  log.info("userPage信息 pages={}", userPage.getPages());
  log.info("userPage信息 total={}", userPage.getTotal());
  log.info("userPage信息 orders={}", userPage.getOrders());
  log.info("userPage信息 records={}", userPage.getRecords());
}
```

**注意：做SQL映射时进行分页**

```java
/**
 * <select id="selectPageVo" resultMapping="BaseResultMap">
 *   SELECT id,name FROM user WHERE state=#{state}
 * </select>
 */
public interface UserMapper extends BaseMapper<User> {
    /**
     * @param page 分页对象,xml中可以从里面进行取值,传递参数 Page 即自动分页,必须放在第一位(你可以继承Page实现自己的分
     */
    IPage<User> selectPageVo(Page<?> page, Integer state);
}
```

### 动态表名

**第一步：配置表名和变量存储类**

```java
/**
 * 存放查询是需要的变量
 */
public class DynamicTableCondition {
    public static ThreadLocal<String> conditionYear = new ThreadLocal<>();
}

/**
 * 添加动态表名插件
 */
@Bean
public MybatisPlusInterceptor paginationInterceptor() {
  MybatisPlusInterceptor mybatisPlusInterceptor = new MybatisPlusInterceptor();
  DynamicTableNameInnerInterceptor dynamicTableNameInnerInterceptor = new DynamicTableNameInnerInterceptor();
  Map<String, TableNameHandler> map = new HashMap<>();
  // 需要进行动态表名支持的表配置
  map.put("user", (sql, tableName) -> tableName + "_" + DynamicTableCondition.conditionYear.get());
  dynamicTableNameInnerInterceptor.setTableNameHandlerMap(map);
  mybatisPlusInterceptor.addInnerInterceptor(dynamicTableNameInnerInterceptor);
  return mybatisPlusInterceptor;
}
```

**第二步：使用动态表名查询**

```java
public void testDynamicTableName() {
  for (int i = 0; i < 6; i++) {
    if (i / 2 == 1) {
      DynamicTableCondition.conditionYear.set("2018");
    } else {
      DynamicTableCondition.conditionYear.set("2019");
    }
    User user = userMapper.selectById(1);
  }
}
```

## 六、Demo工程

[demo工程地址]()