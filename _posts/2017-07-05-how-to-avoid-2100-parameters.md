---
layout: post
title: "用MyBatis望SQLS ERVER里批量插入数据，如何避免超过2100 Parameters错误"
description: "MyBatis,SqlServer"
date: 2017-07-05
tags: [MyBatis,2100 Parameters]
comments: true
share: true
---

如题，方法如下。

1. 计算要插入的每个Record里有多少个参数（Length）。

2. 计算出每次插入的最大件数N（2100/Length）。

3. 使用 Guava's Lists.partition(List, int) method，调用MyBatis的插入批处理。

	List<Item> items = ...
	for (List<Item> partition : Lists.partition(items, 最大件数N)) {
	  // 调用如下的插入批处理
	  this.dao.insert(partition);
	}


	MyBatis的XML:
	<insert id="batchInsertT" parameterType="java.util.List">
		insert into dbo.T (ID, NAME, DEPARTMENT_CD)
		values
		<foreach collection="list" item="value" index="index"
			separator=",">
			(#{value.id,jdbcType=VARCHAR},
			#{value.name,jdbcType=NVARCHAR},
			#{value.departmentCd,jdbcType=VARCHAR}
		</foreach>
	</insert>
---