---
layout: post
title: "spring-data-solr的应用"
date: 2018-12-06 16:07:31
categories: spring
---

# 一、集成spring-data-solr

```xml
<!-- 单机连接 -->
<solr:solr-client id="solrClient" url="http://192.168.1.155:8080/solr" />

<!-- 使用SolrJ的转换器，Spring的转换器对于关联类型不友好 -->
<bean id="solrConverter" class="org.springframework.data.solr.core.convert.SolrJConverter" />

<!-- 配置SolrTemplate，并配置转换器 -->
<bean id="solrTemplate" class="org.springframework.data.solr.core.SolrTemplate">
    <constructor-arg ref="solrClient"/>
    <property name="solrConverter" ref="solrConverter"/>
</bean>
```

# 二、功能的使用

## 1、前置准备

实体类的定义，和使用Solrj一样，注意`Classify`属性

```java
public class Book {
    
    @Field private String id;
    @Field private String name;
    @Field private String issn;
    @Field private List<String> authors;
    @Field private double price;
    private Classify classify;
    @Field private Date pubshingTime;
    @Field private Boolean hasMarc;

    public String getClassifyCode() {
        return this.classify == null ? null : this.classify.getCode();
    }
    
    @Field("classifyCode")
    public void setClassifyCode(String classifyCode) {
        this.classify = (this.classify == null ? new Classify() : this.classify);
        this.classify.setCode(classifyCode);
    }
    //========================其他Getter/Setter========================
}
```

工具类【用于日期转换】

```java
package com.kun.solr.springdata.util;

import java.text.DateFormat;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.time.Instant;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;
import java.util.Date;

public class DateUtil {
    //字符串转换为Date
    static public Date parse(String dateStr) {
        DateFormat formart = new SimpleDateFormat("yyyy-MM-dd");
        Date date = null;
        try {
            date = formart.parse(dateStr);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return date;
    }
    
    //日期类型转换为UTC字符串
    static public String dateToUTCStr(Date date) {
      Instant instant = date.toInstant();
      ZoneId zoneId = ZoneId.systemDefault();
      return instant.atZone(zoneId).format(DateTimeFormatter.ISO_INSTANT);
    }

    //UTC格式字符串转换为Date字符串
    static public String UTCStrToDateStr(String utc) {
        Instant ins = Instant.parse(utc);
        Date date = Date.from(ins);
        DateFormat formart = new SimpleDateFormat("yyyy-MM-dd");
        return formart.format(date);
    }
    
    //Date格式日期转换为字符串
    static public String toDateStr(Date date) {
        DateFormat formart = new SimpleDateFormat("yyyy-MM-dd");
        return formart.format(date);
    }
    
    //Date字符串转UTC格式字符串
    static public String dateStrToUTCStr(String date) {
        return dateToUTCStr(parse(date));
    }
}
```

## 2、增加操作

```java
public void testCreate() throws Exception {
    Book book = new Book();
    book.setId("115373065864171877701");
    book.setIssn("9787302341338");
    book.setName("青春梦想不是梦");
    book.setPrice(23);
    book.setPubshingTime(DateUtil.parse("2016-04-01"));
    book.setAuthors(Arrays.asList("纪广洋"));
    book.setClassifyCode("D");
    book.setHasMarc(true);
    solrTemplate.saveBean("book", book);
    solrTemplate.commit("book");
}
```

## 3、删除操作

```java
public void testDelete() throws Exception {
    solrTemplate.deleteByIds("book", "115373065864171877701");
    solrTemplate.commit("book");
}
```

## 4、查询操作

- 根据ID查询

  ```java
  public void testQueryByID() throws Exception {
        Optional<Book> book = solrTemplate.getById("book","115373065864171873622",Book.class);
        System.out.println(book.get());
    }
  ```

- 普通语法查询

  ```java
  public void testQueryBySyntx() throws Exception {
      Query query = new SimpleQuery("*:*");
      //设置分页信息，不设置getPageable中取值将会报错
      query.setRows(3);
      query.setOffset(7L);
      Page<Book> bookPage = solrTemplate.query("book", query, Book.class);
      System.out.println("总页数：" + bookPage.getTotalPages());	//总页数
      System.out.println("总数：" + bookPage.getTotalElements());	//总元素数
      System.out.println("当前页：" + (bookPage.getPageable().getPageNumber()+1));	//0为基数
      System.out.println("每页大小：" + bookPage.getPageable().getPageSize());		//每页大小
      bookPage.getContent().forEach(System.out::println);
  }
  ```

- 范围查询

  - 时间范围查询

  ```java
  public void testDateRangeQuery() throws Exception {
      //不能直接设置Date类型，需要UTC时区字符串
      String startStr = DateUtil.dateStrToUTCStr("2013-09-01");
      String endStr = DateUtil.dateStrToUTCStr("2013-10-30");
      Criteria criteria = new Criteria("pubshingTime").between(startStr, endStr);
      Query query = new SimpleQuery(criteria);
      //设置分页信息
      query.setRows(5);
      query.setOffset(1L);
      Page<Book> bookPage = solrTemplate.query("book", query, Book.class);
      System.out.println("总页数：" + bookPage.getTotalPages());
      System.out.println("总数：" + bookPage.getTotalElements());
      System.out.println("当前页：" + (bookPage.getPageable().getPageNumber()+1));
      System.out.println("每页大小：" + bookPage.getPageable().getPageSize());
      bookPage.getContent().forEach(System.out::println);
  }
  ```

  - 数字范围查询

  ```java
  public void testNumRangeQuery() throws Exception {
      Criteria criteria = new Criteria("price").between(25, 35);
      Query query = new SimpleQuery(criteria);
      //设置分页信息
      query.setRows(5);
      query.setOffset(0L);
      Page<Book> bookPage = solrTemplate.query("book", query, Book.class);
      System.out.println("总页数：" + bookPage.getTotalPages());
      System.out.println("总数：" + bookPage.getTotalElements());
      System.out.println("当前页：" + (bookPage.getPageable().getPageNumber()+1));
      System.out.println("每页大小：" + bookPage.getPageable().getPageSize());
      bookPage.getContent().forEach(System.out::println);
  }
  ```

- 过滤查询

  ```java
  public void testFilterQuery() throws Exception {
      Query query = new SimpleQuery("*:*");
      //过滤条件
      FilterQuery filterQuery = new SimpleQuery("price:[45 TO *]");
      query.addFilterQuery(filterQuery);
      //设置分页信息
      query.setRows(3);
      query.setOffset(0L);
      Page<Book> bookPage = solrTemplate.query("book", query, Book.class);
      System.out.println("总页数：" + bookPage.getTotalPages());
      System.out.println("总数：" + bookPage.getTotalElements());
      System.out.println("当前页：" + (bookPage.getPageable().getPageNumber()+1));
      System.out.println("每页大小：" + bookPage.getPageable().getPageSize());		
      bookPage.getContent().forEach(System.out::println);
  }
  ```

- 高亮查询

  - 普通高亮查询

    ```java
    @Test
    public void testhighlighQuery() throws Exception {
        //设置查询语句
        Criteria criteria =new Criteria("name").contains("播音");
        HighlightQuery query = new SimpleHighlightQuery(criteria);
        //设置高亮参数
        HighlightOptions hOptions = new HighlightOptions();
        hOptions.addField("name");
        hOptions.setSimplePrefix("<span color='read'>");
        hOptions.setSimplePostfix("</span>");
        //hOptions.setFragsize(10);			//设置段
        query.setHighlightOptions(hOptions);
        //设置分页参数
        query.setRows(5);
        query.setOffset(0L);
        HighlightPage<Book> highlightPage = solrTemplate.queryForHighlightPage("book", query, Book.class);
        //取出高亮信息
        List<Book> books = highlightPage.getContent();
        List<HighlightEntry<Book>> hBookEntrys = highlightPage.getHighlighted();
        for (HighlightEntry<Book> hBookEntry : hBookEntrys) {
            Book book = hBookEntry.getEntity();
            for (Highlight hbook : hBookEntry.getHighlights()) {
                String fieldName = hbook.getField().getName();
                if(fieldName.equals("name")) {
                    book.setName(hbook.getSnipplets().get(0));
                }
            }
        }
        System.out.println(books);
    }
    ```

  - FastVectorHighlighter查询

    ```java
    @Test
    public void testfastVerctorHighlighQuery() throws Exception {
        //设置查询语句
        Criteria criteria =new Criteria("name").contains("播音");
        HighlightQuery query = new SimpleHighlightQuery(criteria);
        //设置高亮参数
        HighlightOptions hOptions = new HighlightOptions();
        hOptions.addField("name");
        hOptions.addHighlightParameter("hl.method", "fastVector");
        hOptions.addHighlightParameter(HighlightParams.TAG_PRE, "<span color='read'>");
        hOptions.addHighlightParameter(HighlightParams.TAG_POST, "</span>");
        //hOptions.setFragsize(10);			//设置段
        query.setHighlightOptions(hOptions);
        //设置分页参数
        query.setRows(5);
        query.setOffset(0L);
        HighlightPage<Book> highlightPage = solrTemplate.queryForHighlightPage("book", query, Book.class);
        //取出高亮信息
        List<Book> books = highlightPage.getContent();
        List<HighlightEntry<Book>> hBookEntrys = highlightPage.getHighlighted();
        for (HighlightEntry<Book> hBookEntry : hBookEntrys) {
            Book book = hBookEntry.getEntity();
            for (Highlight hbook : hBookEntry.getHighlights()) {
                String fieldName = hbook.getField().getName();
                if(fieldName.equals("name")) {
                    book.setName(hbook.getSnipplets().get(0));
                }
            }
        }
        System.out.println(books);
    }
    ```

- 分组查询

  ```java
  @Test
  public void testGroupQuery() throws Exception {
      Query query = new SimpleQuery("*:*");
      //分组参数
      GroupOptions groupOptions = new GroupOptions();
      groupOptions.addGroupByField(new SimpleField("classifyCode"));
      groupOptions.setLimit(2);			//每组最多匹配个数
      groupOptions.setOffset(0);			//每组数据偏移量
      query.setGroupOptions(groupOptions);
      GroupPage<Book> bookGroupPage = solrTemplate.queryForGroupPage("book", query, Book.class);
      //获取分组信息
      GroupResult<Book> groupResult = bookGroupPage.getGroupResult("classifyCode");
      System.out.println("总组数：" + groupResult.getGroupsCount());
      System.out.println("匹配个数：" + groupResult.getMatches());
      for (GroupEntry<Book> book : groupResult.getGroupEntries()) {
          System.out.println(book.getGroupValue());
          book.getResult().forEach(System.out::println);
      }
  }
  ```

- 分面查询

  - 普通字段分片

    ```java
    @Test
    public void testQueryByFacets() throws Exception {
        Criteria criteria = new Criteria();
        criteria.expression("*:*");
        FacetQuery query = new SimpleFacetQuery(criteria);
        FacetOptions facetOptions = new FacetOptions();
        facetOptions.addFacetOnField("classifyCode");
        facetOptions.setFacetLimit(3);				//取几个Facet
        facetOptions.setFacetMinCount(0);			//每组Facet最小有多少元素
        query.setFacetOptions(facetOptions);
        //设置分页信息
        query.setRows(0);
        query.setOffset(0L);
        FacetPage<Book> bookFacetPage = solrTemplate.queryForFacetPage("book", query, Book.class);
        for (Page<FacetFieldEntry> bookFieldPage : bookFacetPage.getFacetResultPages()) {
            System.out.println("分类个数："+bookFieldPage.getTotalElements());
            List<FacetFieldEntry> content = bookFieldPage.getContent();
            for (FacetFieldEntry facetFieldEntry : content) {
            System.out.println(facetFieldEntry.getValue()+":"+facetFieldEntry.getValueCount());
            }
        }
    }
    ```

  - 日期分片

    ```java
    public void testFacetsDateRange() throws Exception {
        Criteria criteria = new Criteria();
        criteria.expression("*:*");
        FacetQuery query = new SimpleFacetQuery(criteria);
        //facets日期查询参数
        FacetOptions facetOptions = new FacetOptions();
        facetOptions.addFacetByRange(new FacetOptions.FieldWithDateRangeParameters(
            "pubshingTime",DateUtil.parse("2013-09-01"),DateUtil.parse("2014-01-01"),"+1MONTH"));
        facetOptions.setFacetMinCount(0);			//每组Facet最小有多少元素
        facetOptions.setFacetSort(FacetSort.COUNT);
        query.setFacetOptions(facetOptions);
    
        //设置分页信息
        query.setRows(0);
        query.setOffset(0L);
        FacetPage<Book> bookFacetPage = solrTemplate.queryForFacetPage("book", query, Book.class);
        //取得信息
        Page<FacetFieldEntry> rangeFacetResultPage = bookFacetPage.getRangeFacetResultPage("pubshingTime");
        System.out.println("总个数：" + rangeFacetResultPage.getTotalElements());
        for (FacetFieldEntry facetFieldEntry : rangeFacetResultPage) {
            System.out.println(facetFieldEntry.getValue() + ":" + facetFieldEntry.getValueCount());
        }
    }
    ```

  - 数字分片

    ```java
    @Test
    public void testFacetsNumRange() throws Exception {
        Criteria criteria = new Criteria();
        criteria.expression("*:*");
        FacetQuery query = new SimpleFacetQuery(criteria);
        FacetOptions facetOptions = new FacetOptions();
        facetOptions.addFacetByRange(new FacetOptions.FieldWithNumericRangeParameters(
            "price",20,50,10));
        facetOptions.setFacetMinCount(0);			//每组Facet最小有多少元素
        query.setFacetOptions(facetOptions);
        //设置分页信息
        query.setRows(0);
        query.setOffset(0L);
        //读取信息
        FacetPage<Book> bookFacetPage = solrTemplate.queryForFacetPage("book", query, Book.class);
        Page<FacetFieldEntry> rangeFacetResultPage = bookFacetPage.getRangeFacetResultPage("price");
        System.out.println("总格数：" + rangeFacetResultPage.getTotalElements());
        for (FacetFieldEntry facetFieldEntry : rangeFacetResultPage) {
            System.out.println(facetFieldEntry.getValue() +":"+ facetFieldEntry.getValueCount());
    
        }
    }
    ```

- suggest查询

  ```java
  @Test
  public void testSuggestQuery() throws Exception {
      SolrClient solrClient = solrTemplate.getSolrClient();
      SolrQuery query = new SolrQuery();
      //请求参数封装
      query.setRequestHandler("/suggest");				//handler
      query.add("suggest","true");						//必须设置为true
      query.add("suggest.dictionary","issnSuggester");	//词典
      query.add("suggest.q","9");							//查询参数
      query.add(CommonParams.WT,"json");					//返回格式
      query.add("suggest.build","true"); 
      query.add("suggest.cfq","D");						//根据上下文过滤
      QueryResponse response = solrClient.query("book",query);
      //获得建议的结果
      SuggesterResponse suggesterResponse = response.getSuggesterResponse();
      Map<String, List<String>> suggestedTerms = suggesterResponse.getSuggestedTerms();
      List<String> termList = suggestedTerms.get("issnSuggester");
      System.out.println(termList);
  }
  ```

  详情参考[链接](https://mynamelancelot.github.io/searchengine/solr.html)

- moreLikeThis查询

  ```java
  @Test
  public void testMoreLikeThisQuery() throws Exception {
      SolrClient solrClient = solrTemplate.getSolrClient();
      SolrQuery query = new SolrQuery("*:*");			//查询，返回的每个结果都会进行相似匹配
      query.setFields("id");							//返回的field
      query.addMoreLikeThisField("classifyCode");		//判读相似的字段
      query.setMoreLikeThisCount(2);					//返回相似文档信息的个数
      query.setMoreLikeThisMinDocFreq(1);				//最小文档频率，所在文档的个数小于这个值的词将不用于相似判断
      query.setMoreLikeThisMinTermFreq(1);			//最小分词频率，在单个文档中出现频率小于这个值的词将不用于相似判断
      QueryResponse response = solrClient.query("book", query);
      NamedList<SolrDocumentList> moreLikeThisList = response.getMoreLikeThis();
      for (Entry<String, SolrDocumentList> entry : moreLikeThisList) {
          System.out.println("id:" + entry.getKey());
          SolrDocumentList documentList = entry.getValue();
          System.out.println("相似总个数:" + documentList.getNumFound());
          for (SolrDocument solrDocument : documentList) {
              System.out.println(solrDocument.get("id"));
          }
      }
  }
  ```

  详情参考[链接](https://mynamelancelot.github.io/searchengine/solr.html)
  

