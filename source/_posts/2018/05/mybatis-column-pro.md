---
title: mybatis中association和collection的column传入多个参数值
date: 2018-05-30 23:43:58
tags: [association, collection]
categories: [mybatis]
---




## mybatis中association和collection的column传入多个参数值

项目中在使用association和collection实现一对一和一对多关系时需要对关系中结果集进行筛选，如果使用懒加载模式，即联合使用`select`标签时，主sql和关系映射里的sql是分开的，查询参数传递成为问题。

**mybatis文档:**  

| property | description    |
| :------------- | :------------- |
| column     | 数据库的列名或者列标签别名。与传递给resultSet.getString(columnName)的参数名称相同。注意： 在处理组合键时，您可以使用column=“{prop1=col1,prop2=col2}”这样的语法，设置多个列名传入到嵌套查询语句。这就会把prop1和prop2设置到目标嵌套选择语句的参数对象中。|

<!-- more -->

### 实例

```xml
<resultMap id="findCountryCityAddressMap" type="map">
    <result property="country" column="country"/>
    <collection property="cityList"
          column="{cityId=city_id,adr=addressCol, dis=districtCol}" //adr作为第二个sql查询条件key,即prop1属性
          ofType="map"                                              //addressCol即为虚拟列名
          javaType="java.util.List" select="selectAddressByCityId"/>
</resultMap>

<resultMap id="selectAddressByCityIdMap" type="map">
    <result property="city" column="city"/>
    <collection property="addressList" column="city" ofType="map" javaType="java.util.List">
        <result property="address" column="address"/>
        <result property="district" column="district"/>
    </collection>
</resultMap>

<select id="findCountryCityAddress" resultMap="findCountryCityAddressMap">
    SELECT
        ct.country,
        ci.city_id,
        IFNULL(#{addressQuery},'') addressCol, //为传入查询条件,构造虚拟列，虚拟列为查询条件参数值
        IFNULL(#{districtQuery},'') districtCol
    FROM
        country ct
    LEFT JOIN city ci ON ct.country_id = ci.country_id
    ORDER BY ct.country_id
</select>

<select id="selectAddressByCityId" parameterType="java.util.Map" resultMap="selectAddressByCityIdMap">
    SELECT
        ci.city,
        ads.address,
      ads.district
    FROM
        (
            SELECT
                city,
                city_id
            FROM
                city ci
            WHERE
                ci.city_id = #{cityId}
        ) ci
    LEFT JOIN address ads ON ads.city_id = ci.city_id
    <where>
        <if test="adr!=null and adr!=''">
            and ads.address RegExp #{adr}
        </if>
        <if test="dis!=null and dis!=''">
            ads.district Regexp #{dis}
        </if>
    </where>

</select>
```


测试文件:  

```java
@Test
public void findCountryCityAddressTest() throws JsonProcessingException {
    Map<String,Object> param = new HashMap<>();
    param.put("addressQuery","1168");
    List<Map<String, Object>> rs = countryManager.findCountryCityAddress(param);
    ObjectMapper mapper = new ObjectMapper();
    ObjectWriter writer = mapper.writerWithDefaultPrettyPrinter();
    System.out.println(writer.writeValueAsString(rs));
}
```


测试结果:  

```json
[
	{
		"country": "Afghanistan",
		"cityList": [{
				"city": "Kabul",
				"addressList": [{
						"address": "1168 Najafabad Parkway",
						"district": "Kabol"
					}
				]
			}
		],
		"city_id": 251
	},
	{
		"country": "Algeria",
		"cityList": [],
		"city_id": 59
	}
]
```

可以看到，确实将查询条件通过column参数传入到第二个sql中，并执行成功
