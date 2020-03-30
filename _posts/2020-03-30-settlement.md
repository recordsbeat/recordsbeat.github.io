---
title: "본격 정산모듈작성"
date: 2020-03-30 19:37:00 -0400
categories: mybatis resultMap
---

갓배민의 이동욱님이 지은

'스프링 부트와 AWS로 혼자 구현하는 웹서비스'를 읽으며 업무를 병행 중..

이 책에서 사용하는 개발환경은 spring boot 와 jpa를 사용한다.

하지만 나의 업무환경은 spring boot와 mybatis ( IDE도 다르지만 이건 그냥 넘어갈 수 있는 수준)

이 책과 나의 실 환경의 괴리는 이쯤에서 시작한다.

JPA 환경에서 필요로하는 Entity 그리고 통신을 위한 DTO 이 둘은 Mybatis 환경에서 굳이 필요할까? 라는 생각을 하게 만들었다.

직접 질의를 작성하는 마당에 원하는 칼럼들을 뽑아올 수 있고, Entity 클래스를 만들어야하는지 혼란스러웠다.

하지만, 나중에 JPA로 넘어갈 것을 염두해두고 작성을 시작

지금 나의 개발 계획은 아래와 같다.

1.mysql 상의 view를 통해 원하는 데이터를 볼 수 있도록 미리 join해둔다.<br>
2.view를 사용하여 질의를 작성하고 이를 도메인으로 DB와 통신한다.(Entity처럼..)<br>
3.Entity와 DTO를 서비스에서 컨버팅하여 view페이지와 통신하도록 한다.<br>

위의 계획을 토대로하여 mysql view를 만들었고, 이를 받아들일 도메인을 작성하였다.


```

import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.ToString;

@NoArgsConstructor
@Getter
@ToString
public class Reservation {
	private int resIdx;
	...
	private City city;
	private Settlement settlement;

}

```
사실 상당히 긴 도메인을 작성하게 되었다. 이유는 다른 DTO로도 컨버팅이 가능하게끔 되도록 공통되는 데이터를 끌어오고 싶었기 때문에<br>
(소위 말하는 통DB가 되어버린 느낌?)

해당 도메인과 mybatis가 매핑이 되려면 친절히 칼럼과 필드를 매칭 시켜주어야한다.


아래와 같이 resultMap을 사용하여 도메인 매핑을 진행, 정상적으로 매핑된것을 확인해였다.

참고링크 
https://github.com/sujini1002/shopmall/wiki/MyBatis%EC%97%90%EC%84%9C-ResultMap%EA%B3%BC-Map%EC%9C%BC%EB%A1%9C-%EA%B2%B0%EA%B3%BC%EB%B0%9B%EA%B8%B0



```
	<resultMap type="com.guivingAdmin.domain.City" id="cityResult">
		<result property="city" column="CITY_IDX"/>
		<result property="country" column="COUNTRY_IDX"/>			
		<result property="currency" column="CITY_CURRENCY"/>
	</resultMap>
	<resultMap type="com.guivingAdmin.domain.Settlement" id="settlementResult">
		<result property="resIdx" column="RES_IDX"/>
		<result property="status" column="SETTLE_STATUS"/>			
		<result property="trTime" column="SETTLE_TIME"/>
	</resultMap>
	
	<resultMap type="com.guivingAdmin.domain.Reservation" id="reservationResult">
		<result property="resIdx" column="RES_IDX"/>
		...
    	<collection property="city" resultMap="cityResult"/>
    	<collection property="settlement" resultMap="settlementResult"/>
	</resultMap>
```


그리고 또 하나. 위의 ResultMap은 앞으로 주구장창 사용할거 같아서 찾아보니 다음 링크가 도움이 되었다.


참고링크 
https://deoki.tistory.com/55

공통으로 사용될 ResultMap을 미리 설정해 두고 dao를 통하여 패키지처럼 불러온다.


```
package com.guivingAdmin.dao;

public interface ResultMap {
}
```


```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.guivingAdmin.dao.ResultMap">

	<resultMap type="com.guivingAdmin.domain.City" id="cityResult">
		<result property="city" column="CITY_IDX"/>
		<result property="country" column="COUNTRY_IDX"/>			
		<result property="currency" column="CITY_CURRENCY"/>
	</resultMap>
	<resultMap type="com.guivingAdmin.domain.Settlement" id="settlementResult">
		<result property="resIdx" column="RES_IDX"/>
		<result property="status" column="SETTLE_STATUS"/>			
		<result property="trTime" column="SETTLE_TIME"/>
	</resultMap>
	
	<resultMap type="com.guivingAdmin.domain.Reservation" id="reservationResult">
		...
    	<collection property="city" resultMap="cityResult"/>
    	<collection property="settlement" resultMap="settlementResult"/>
	</resultMap>
</mapper>
```

대망의 쿼리

```
<select id="selectAllSettlement" resultMap="com.guivingAdmin.dao.ResultMap.reservationResult">
		SELECT 
			...
		FROM RD,ST
		WHERE RD.RES_IDX = ST.RES_IDX
		ORDER BY RES_DATE DESC, RES_IDX DESC
	</select>
```
위와 같이 resultMap의 경로를 패키지처럼 불러와 주면 정상 매핑 처리 완료

아직 DTO로 컨버팅하여 통신하는 부분은 미작성 내일 마저해야짓

그 다음은 페이지 작업과 searchDTO와 Paging 동시에 어떻게 할지 생각해보기

javascript 공통 js 부분 추출 및 적용해보기 .class적극활용해서

같은 검색 필터 컴포넌트끼리는 작성하지 않도록 (페이지별 인터페이스처럼 사용할 수 있는 환경이 갖춰지면 더 좋을듯)



