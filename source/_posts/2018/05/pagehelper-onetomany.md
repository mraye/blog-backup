---
title: 'mybatis中使用pageHelper实现一对多分页'
date: 2018-05-26 21:32:07
tags: [mybatis, pageHelper]
categories: [mybatis]
---



### mybatis中使用pageHelper实现一对多分页

项目里经常遇到一对多分页的问题，以前认为PageHelper不支持一对多的分页，还一直自己重写分页方法。。。。

#### 数据库表

+ country表

```sql
CREATE TABLE `country` (
  `country_id` smallint(5) unsigned NOT NULL AUTO_INCREMENT,
  `country` varchar(50) NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`country_id`)
) ENGINE=InnoDB AUTO_INCREMENT=110 DEFAULT CHARSET=utf8;


```

+ city表

<!-- more -->

```sql
CREATE TABLE `city` (
  `city_id` smallint(5) unsigned NOT NULL AUTO_INCREMENT,
  `city` varchar(50) NOT NULL,
  `country_id` smallint(5) unsigned NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`city_id`),
  KEY `idx_fk_country_id` (`country_id`),
  CONSTRAINT `fk_city_country` FOREIGN KEY (`country_id`)
  REFERENCES `country` (`country_id`) ON UPDATE CASCADE
) ENGINE=InnoDB AUTO_INCREMENT=601 DEFAULT CHARSET=utf8;

```

#### pageHelper配置

使用gradle依赖:  

```gradle
dependencies {

    /* omit... */
    testCompile group: 'junit', name: 'junit', version: '4.12'
    compile 'com.github.pagehelper:pagehelper:5.1.4'
}

```
在`spring-context.xml`配置`pageHelper`插件：  

```xml
<bean id="sessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
     <property name="dataSource" ref="dataSource"/>
     <property name="typeAliasesPackage" value="com.github.**.domain"/>
     <property name="configLocation" value="classpath:mybatis-config.xml"/>
     <property name="mapperLocations" value="classpath:mappings/**/*.xml"/>
     <property name="plugins">
         <array>
             <bean class="com.github.pagehelper.PageInterceptor">
                 <!-- 这里的几个配置主要演示如何使用，如果不理解，一定要去掉下面的配置 -->
                 <property name="properties">
                     <value>
                         helperDialect=mysql
                         reasonable=true
                         autoRuntimeDialect=true
                     </value>
                 </property>
             </bean>
         </array>
     </property>
 </bean>
```


`countryDo`和`cityDo`:


```java

public class CityDo implements Serializable {
    private static final long serialVersionUID = -1453407185811057526L;
    private Long cityId;
    private Long countryId;
    private String city;
    private String lastUpdate;
    // ...omit getter and setter
}

public class CountryDo implements Serializable {
    private static final long serialVersionUID = 7914312649965400868L;
    private Long countryId;
    private String country;
    private String lastUpdate;
    // ...omit getter and setter
}

```


`countryManager.java`文件:  

```java
@Component("countryManager")
public class CountryManager {

    @Autowired
    private CountryDao countryDao;

    public List<Map<String, Object>> findCountryList(Map<String, Object> map){
        PageInfo<Map<String, Object>> page = null;
        PageHelper.startPage(
                map.get("pageIndex")==null ? 1 : Integer.valueOf(map.get("pageIndex").toString()),
                map.get("pageSize")==null ? 2 : Integer.valueOf(map.get("pageSize").toString())
        );

        List<Map<String, Object>> list = countryDao.findCountryList(map);
        page = new PageInfo<>(list);
        return page.getList();
    }

}
```

mapper映射文件`CountryDao.xml`,**所有一对多关系都是懒加载的**:  

```xml
<resultMap id="findCountryListMap" type="map">
    <result property="countryId" column="countryId"/>
    <result property="country" column="country"/>
    <collection property="cityList" column="countryId" ofType="map" javaType="java.util.List"
			    select="getCityByCountryId">
        <result property="city" column="city"/>
        <result property="cityId" column="cityId"/>
    </collection>
</resultMap>

<select id="getCityByCountryId" parameterType="long" resultType="map">
    SELECT
        ci.city,
        ci.city_id cityId
    FROM
        city ci
    WHERE
        //countryId这里传值进来的是 findCountryList中countryId的列名，
        //即findCountryListMap中的column属性而不是property属性名
        ci.country_id=#{countryId}
    order by ci.city_id
</select>

<select id="findCountryList" resultMap="findCountryListMap">
    SELECT
        cy.country_id countryId,
        cy.country
    FROM
        country cy
    ORDER BY cy.country_id
</select>
```


测试案例:  

```java

@Test
public void findCountryListTest() throws JsonProcessingException {
    Map<String, Object> map = new HashMap<>();
    map.put("pageIndex",1);
    map.put("pageSize",2);
    List<Map<String, Object>> result = countryManager.findCountryList(map);
    ObjectMapper mapper = new ObjectMapper();
    ObjectWriter writer = mapper.writerWithDefaultPrettyPrinter();
    System.out.println("start page(1,2)");
    System.out.println(writer.writeValueAsString(result));
    System.out.println("============================");
    System.out.println("start page(2,2)");
    map.put("pageIndex",2);
    result = countryManager.findCountryList(map);
    System.out.println(writer.writeValueAsString(result));
}
```


测试结果:  

```json
start page(1,2):  第一页
-------------------------------------------
[
	{
		"countryId": 1
		"country": "Afghanistan",
		"cityList": [{
				"city": "Kabul",
				"cityId": 251
			}
		],
	},
	{
		"countryId": 2
		"country": "Algeria",
		"cityList": [{
				"city": "Batna",
				"cityId": 59
			}, {
				"city": "Bchar",
				"cityId": 63
			}, {
				"city": "Skikda",
				"cityId": 483
			}
		],
	}
]
-------------------------------------------

start page(2,2): 第2页
-------------------------------------------
[
	{
		"countryId": 3
		"country": "American Samoa",
		"cityList": [{
				"city": "Tafuna",
				"cityId": 516
			}
		],
	},
	{
		"countryId": 4
		"country": "Angola",
		"cityList": [{
				"city": "Benguela",
				"cityId": 67
			}, {
				"city": "Namibe",
				"cityId": 360
			}
		],
	}
]
-------------------------------------------
```
