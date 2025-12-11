---
title: 深入理解Mybatis
tags:
  - mybatis
  - mysql
categories: 编程
date: 2024-12-18 19:04:25
---

# 深入理解Mybatis(常见属性)

- 序言: 本文是对mybatis中的字段回显到实体类(属性, 数组, 哈希表)的方法总结

## 类映射resultType

- 最常见的JavaBean映射, 就是通过类型别名`typeAliases`来处理, 需要在配置文件中指定实体类路径

```java
type-aliases-package: com.core.entity # 别名扫描包
// 然后再xml文件中就可以省略具体路径
<select id="getAdmin" resultType="umsAdmin">
```

- 特点: **返回的结果自动映射到实体类Type上, 要求字段要完全一致, 不支持修改字段名**

## 字段映射resultMap

```xml
<resultMap id="adminMap" type="custmeAdmin">
        <result column="id" property="custmeId"></result>
</resultMap>
```

```xml
// 使用方法
<select id="getAdmin" resultMap="adminMap">
```

- 这里面`column`对应的是数据库的列名或别名；`property`对应的是结果集的字段或属性
- **特点: 自定义映射字段灵活性较高, 但是实现复杂(需要自己手写字段对应关系)**

## 联合association

- 当resultMap中有字段是bean类嵌套时, 需要通过association去嵌套编写对应关系  **private Role role;**

```xml
<resultMap id="userMap" type="umsAdmin">
	<id property="id" column="id"></id>
	<result property="username" column="username"></result>
	<result property="password" column="password"></result>
	<result property="address" column="address"></result>
	<result property="email" column="email"></result>
	
	<association property="role" javaType="Role">  
		<id property="id" column="role_id"></id>
		<result property="name" column="role_name"></result>
	</association>
</resultMap>
{
    "id": "1001",
    "username": "后羿",
    "password": "123456",
    "address": "北京市海淀区",
    "email": "510273027@qq.com",
    "role": {
        "id": "3",
        "name": "射手"
    }
}
```

- **特点: 在resultMap中嵌套实体类, 不支持再嵌套实体类数组**

## 集合collection

- 当resultMap中属性为数组形式时, 可拓展…  **private List< Role > roleList;**
- **主要的属性: property(实体类数组名)  ofType(实体类数组类型名,一定要写)**

```xml
<resultMap id="userMap" type="User">
	<id property="id" column="id"></id>
	<result property="username" column="username"></result>
	<result property="password" column="password"></result>
	<result property="address" column="address"></result>
	<result property="email" column="email"></result>
	
	<collection property="roles" ofType="Role">
		<id property="id" column="role_id"></id>
		<result property="name" column="role_name"></result>
	</collection>
</resultMap>
{
    "id": "1003",
    "username": "貂蝉",
    "password": "123456",
    "address": "北京市东城区",
    "email": "510273027@qq.com",
    "roles": [
        {
            "id": "1",
            "name": "中单"
        },
        {
            "id": "2",
            "name": "打野"
        }
    ]
}
```

- **特点: 支持在resultMap中嵌套数组遍历映射, 是association的高级版**

## 例子

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.admin.mapper.productMapper">
	<!--左连接查询 分主表和次表 通过自定义resultMap对结果集进行映射-->  <----------
    <select id="updateInfo" resultMap="productMap">
        SELECT p.*,
               pc.parent_id as        cate_parent_id,
               l.id                   ladder_id,
               l.product_id           ladder_product_id,
               l.discount             ladder_discount,
               l.count                ladder_count,
               l.price                ladder_price,
               pf.id                  full_id,
               pf.product_id          full_product_id,
               pf.full_price          full_full_price,
               pf.reduce_price        full_reduce_price,
               m.id                   member_id,
               m.product_id           member_product_id,
               m.member_level_id      member_member_level_id,
               m.member_price         member_member_price,
               m.member_level_name    member_member_level_name,
               s.id                   sku_id,
               s.product_id           sku_product_id,
               s.price                sku_price,
               s.promotion_price      sku_promotion_price,
               s.low_stock            sku_low_stock,
               s.pic                  sku_pic,
               s.sale                 sku_sale,
               s.sku_code             sku_sku_code,
               s.stock                sku_stock,
               s.sp_data              sku_sp_data,
               a.id                   attribute_id,
               a.product_id           attribute_product_id,
               a.product_attribute_id attribute_product_attribute_id,
               a.value                attribute_value
        FROM pms_product p
                 LEFT JOIN pms_product_category pc on pc.id = p.product_category_id
                 LEFT JOIN pms_product_ladder l ON p.id = l.product_id
                 LEFT JOIN pms_product_full_reduction pf ON pf.product_id = p.id
                 LEFT JOIN pms_member_price m ON m.product_id = p.id
                 LEFT JOIN pms_sku_stock s ON s.product_id = p.id
                 LEFT JOIN pms_product_attribute_value a ON a.product_id = p.id
        WHERE p.id = #{id};
    </select>
	<!--最大的结果集类叫productVO 包含了主表product全部属性和cateParentId-->  <----------
    <resultMap id="productMap" type="com.admin.pojo.vo.productVO">
		<!--额外属性-->
        <result property="cateParentId" column="cate_parent_id"/>
		<!--主表属性-->
        <result property="brandId" column="brand_id"/>
        <result property="productCategoryId" column="product_category_id"/>
        <result property="feightTemplateId" column="feight_template_id"/>
        <result property="productAttributeCategoryId" column="product_attribute_category_id"/>
        <result property="name" column="name"/>
        <result property="pic" column="pic"/>
        <result property="productSn" column="product_sn"/>
        <result property="deleted" column="deleted"/>
        <result property="publishStatus" column="publish_status"/>
        <result property="newStatus" column="new_status"/>
        <result property="recommandStatus" column="recommand_status"/>
        <result property="verifyStatus" column="verify_status"/>
        <result property="sort" column="sort"/>
        <result property="sale" column="sale"/>
        <result property="price" column="price"/>
        <result property="promotionPrice" column="promotion_price"/>
        <result property="giftGrowth" column="gift_growth"/>
        <result property="giftPoint" column="gift_point"/>
        <result property="usePointLimit" column="use_point_limit"/>
        <result property="subTitle" column="sub_title"/>
        <result property="originalPrice" column="original_price"/>
        <result property="stock" column="stock"/>
        <result property="lowStock" column="low_stock"/>
        <result property="unit" column="unit"/>
        <result property="weight" column="weight"/>
        <result property="previewStatus" column="preview_status"/>
        <result property="serviceIds" column="service_ids"/>
        <result property="keywords" column="keywords"/>
        <result property="note" column="note"/>
        <result property="albumPics" column="album_pics"/>
        <result property="detailTitle" column="detail_title"/>
        <result property="promotionStartTime" column="promotion_start_time"/>
        <result property="promotionEndTime" column="promotion_end_time"/>
        <result property="promotionPerLimit" column="promotion_per_limit"/>
        <result property="promotionType" column="promotion_type"/>
        <result property="brandName" column="brand_name"/>
        <result property="productCategoryName" column="product_category_name"/>
        <result property="description" column="description"/>
        <result property="detailDesc" column="detail_desc"/>
        <result property="detailHtml" column="detail_html"/>
        <result property="detailMobileHtml" column="detail_mobile_html"/>
		<!--次表数组  List<PmsProductLadder> productLadderList;-->
        <collection property="productLadderList" columnPrefix="ladder_" ofType="pmsProductLadder">
            <id column="id" jdbcType="BIGINT" property="id"/>
            <result column="product_id" jdbcType="BIGINT" property="productId"/>
            <result column="count" jdbcType="INTEGER" property="count"/>
            <result column="discount" jdbcType="DECIMAL" property="discount"/>
            <result column="price" jdbcType="DECIMAL" property="price"/>
        </collection>
        <collection property="memberPriceList" ofType="pmsMemberPrice">
            <id column="member_id" jdbcType="BIGINT" property="id"/>
            <result column="member_product_id" property="productId"/>
            <result column="member_member_level_id" property="memberLevelId"/>
            <result column="member_member_price" property="memberPrice"/>
            <result column="member_member_level_name" property="memberLevelName"/>
        </collection>
        <collection property="productAttributeValueList" columnPrefix="attribute_" ofType="pmsProductAttributeValue">
            <id column="id" jdbcType="BIGINT" property="id"/>
            <result column="product_attribute_id" property="productAttributeId"/>
            <result column="product_id" property="productId"/>
        </collection>
        <collection property="skuStockList" columnPrefix="sku_" ofType="pmsSkuStock">
            <id column="id" jdbcType="BIGINT" property="id"/>
            <result column="product_id" jdbcType="BIGINT" property="productId"/>
            <result column="sku_code" jdbcType="VARCHAR" property="skuCode"/>
            <result column="price" jdbcType="DECIMAL" property="price"/>
            <result column="stock" jdbcType="INTEGER" property="stock"/>
            <result column="low_stock" jdbcType="INTEGER" property="lowStock"/>
            <result column="pic" jdbcType="VARCHAR" property="pic"/>
            <result column="sale" jdbcType="INTEGER" property="sale"/>
            <result column="promotion_price" jdbcType="DECIMAL" property="promotionPrice"/>
            <result column="lock_stock" jdbcType="INTEGER" property="lockStock"/>
            <result column="sp_data" jdbcType="VARCHAR" property="spData"/>
        </collection>
        <collection property="productFullReductionList" columnPrefix="full_" ofType="pmsProductFullReduction">
            <id column="id" jdbcType="BIGINT" property="id"/>
            <result column="product_id" jdbcType="BIGINT" property="productId"/>
            <result column="full_price" jdbcType="DECIMAL" property="fullPrice"/>
            <result column="reduce_price" jdbcType="DECIMAL" property="reducePrice"/>
        </collection>
		<!--这两条是相关子查询 根据父查询的结果条件来查询-->
        <collection property="subjectProductRelationList" column="{productId=id}"
                    select="selectSubjectProductRelationByProductId" ofType="cmsSubjectProductRelation"/>
        <collection property="prefrenceAreaProductRelationList" column="{productId=id}"
                    select="selectPrefrenceAreaProductRelationByProductId" ofType="cmsPrefrenceAreaProductRelation"/>
    </resultMap>

	<!--子查询的语句-->
    <select id="selectSubjectProductRelationByProductId"
            resultMap="SubjectProductRelationByProductIdMap">
        select *
        from cms_subject_product_relation
        where product_id = #{productId}
    </select>
    <select id="selectPrefrenceAreaProductRelationByProductId"
            resultMap="PrefrenceAreaProductRelationByProductIdMap">
        select *
        from cms_prefrence_area_product_relation
        where product_id = #{productId}
    </select>
	<!--子查询的resultMap结果容器  List<CmsSubjectProductRelation> subjectProductRelationList -->
    <resultMap id="SubjectProductRelationByProductIdMap" type="CmsSubjectProductRelation">
        <id column="id" jdbcType="BIGINT" property="id"/>
        <result column="subject_id" jdbcType="BIGINT" property="subjectId"/>
        <result column="product_id" jdbcType="BIGINT" property="productId"/>
    </resultMap>
    <resultMap id="PrefrenceAreaProductRelationByProductIdMap" type="CmsPrefrenceAreaProductRelation">
        <id column="id" jdbcType="BIGINT" property="id"/>
        <result column="prefrence_area_id" jdbcType="BIGINT" property="prefrenceAreaId"/>
        <result column="product_id" jdbcType="BIGINT" property="productId"/>
    </resultMap>
</mapper>
```

## 为什么使用雪花算法生成id
- id生成类型:
	- 自增: 适用于单体架构单一数据库, 如果使用分库会出现id重复
	- UUID: 本身是无序的, 又可能出现重复, 从来一角度来说不断增长的id更适和创建索引
	- 雪花id构成((bit)): 1位不用 + 41位时间戳(依据本身特性->自增的) + 10位机器位(分库, 不同机器从同一个Redis中拿到一个机器码) + 12位保障位(同一毫秒内有4096个id)
- 雪花优点: 全局唯一, 有序增长, 生成效率高