---
title: "로드하라 N+1 을 해결하라"
date: 2020-04-07 19:55:00 -0400
categories: mybatis collection oneToMany
---
<br>
어제 시도해보았던 mybatis를 통한 related Object load 하기<br>
<br>
대략적으로 resultMap을 정해주고 거기에 데이터를 매핑해주면 되는 직관적인 방식<br>
<br>
lazy load를 걸기 위해 이리저리 찾아보았고 N+1의 이슈는 lazy load로만은 해결 할 수 없다는 것을 알게 되었다.<br>
<br>
Mybatis를 어느정도 ORM에 가깝게 사용하려다 보니 나타나는 부작용. 알고보니 JPA에서도 나타나는 이슈다.<br>
<br>
참고링크<br>
<br>
https://deveric.tistory.com/74<br>
<br>
위 링크는 상당히 설명을 잘해주었다.<br>
<br>
as-is 가 현 회사의 코드와 비슷한 측면이 있고 (객체를 순환 탐색하며 db에서 데이터를 꺼내오는)<br>
<br>
이는 이전부터 잘못되었다고 느낀 부분이다.<br>
<br>
이 글에서 정리하는 바는 이렇다.<br>
<br>
resultMap과 Outer join을 사용하여 매핑할 객체의 데이터들을 한번의 쿼리로 불러온다.<br>
<br>
(잉? 이거 어디서 많이본 듯)<br>
<br>
이전에 내가 mysql view를 만들어서 하나씩 매핑시켰던 방식과 동일하다.<br>
<br>
엉겁결에 얻어걸린 방식이지만 해당 방법이 N+1 이슈를 최소화할 수 있는 방법이라고 한다.<br>
<br>
곰곰히 생각해보니 맞기도하다. left join 이라고 했을때 좌측에는 메인되는 객체의 데이터가 오른쪽에는 필드(related Object)객체의 데이터가<br>
<br>
나올 것이다. 이는 lazy load를 사용했을 때 쿼리요청이 get의 횟수만큼 발생하는 것과 차이가 있어보인다.<br>
<br>
또한 resultMap 사용시 각 resultMap에 id되는 값을 할당하면 중복되는 데이터는 피하여 탐색한다고 한다.(좋네..)<br>
<br>
<br>
다음 참고링크<br>
<br>
https://lyb1495.tistory.com/110<br>
<br>
<br>
사실 내가 원하던 TO-BE방식에 가장 근접한 설명을 해주었다.<br>
<br>
도메인은 관련된 객체를 필드로 가지며 비지니스 로직을 처리할 수 있도록 하는 것.<br>
<br>
하지만 위 글에서도 설명하듯 몇가지 문제점이 따른다.<br>
<br>
-resultMap과 SQL문이 중복된다.<br>
resultMap의 중복을 최대한 피하기 위해 공통적으로 컬럼명을 사용하도록 하였고 nested형태를 취하는 resultMap을 각각 선언해두었다.<br>
<br>
-1:N , N:1 관계 도메인 데이터의 상이함.<br>
가령 1:N 관계에서 N의 총 합을 구한다고 할 때 1이 되는 객체는 N의 갯 수를 파악해서 합을 구할 것이다.<br>
<br>
그러나 N이 되는 객체의 필드로 1객체가 존재한다고 할때 N의 총 합을 구하는 메소드는 정상적으로 작동하지 않을 수 있다.<br>
<br>
이 부분은 당장 개발에 적용하지 않은 부분이지만 위에서 언급한 것과 같이 resultMap자체에 nested 형태를 취하지 않고 별 개의 resultMap형태로<br>
<br>
설정해 놓았기 때문에 아직까지는 문제로 보이진 않는다.<br>
<br>

ResultMap
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="ResultMap">

	<resultMap type="com.guivingAdmin.domain.City" id="city">
		<id property="city" column="CITY_IDX"/>
		<result property="country" column="COUNTRY_IDX"/>			
		<result property="currency" column="CITY_CURRENCY"/>		
		<result property="cityCode" column="CITY_CODE"/>		
		<result property="cityInService" column="CITY_IN_SERVICE"/>		
		<result property="cityUtc" column="CITY_UTC"/>
	</resultMap>
	
	<resultMap type="com.guivingAdmin.domain.Settlement" id="settlement">
		<id property="resIdx" column="RES_IDX"/>
		<result property="status" column="STATUS"/>				
		<result property="trTime" column="TR_TIME"/>	
		<result property="gdSettle" column="GD_SETTLE"/>			
		<result property="gotSettle" column="GOT_SETTLE"/>
		<result property="godSettle" column="GOD_SETTLE"/>	
		<collection property="reservation" resultMap="reservation"/>
		<!-- 
		<collection property="reservation"  fetchType="lazy" column="RES_IDX"  select="com.guivingAdmin.dao.ReservationMapper.selectReservationByResIdx"/>
		 -->
	</resultMap>
	
	<resultMap type="com.guivingAdmin.domain.Company" id="company">
		<id property="comIdx" column="COM_IDX"/>
		<result property="comName" column="COM_NAME"/>			
		<result property="comOwnerName" column="COM_OWNER_NAME"/>
		<result property="comBuildDate" column="COM_BUILD_DATE"/>			
		<result property="comBizNum" column="COM_BIZ_NUM"/>
		<result property="comAddr" column="COM_ADDR"/>			
		<result property="comBizLicenseUrl" column="COM_BIZ_LICENSE_URL"/>
		<result property="comAuthCode" column="COM_AUTH_CODE"/>			
		<result property="comAddrCity" column="COM_ADDR_CITY"/>	
		<result property="comAddrState" column="COM_ADDR_STATE"/>	
    	<collection property="citys" resultMap="city"/>
	</resultMap>
	
	<resultMap type="com.guivingAdmin.domain.User" id="user">		
		<id property="userIdx" column="USER_IDX"/>
		<result property="userUid" column="USER_UID"/>
		<result property="userEmail" column="USER_EMAIL"/>
		<result property="userType" column="USER_TYPE"/>
		<result property="userPwd" column="USER_PWD"/>
		<result property="userFName" column="USER_F_NAME"/>
		<result property="userLName" column="USER_L_NAME"/>
		<result property="userPhoneNum" column="USER_PHONE_NUM"/>
		<result property="userImageUrl" column="USER_IMAGE_URL"/>
		<result property="userNation" column="USER_NATION"/>
		<result property="userLanguage" column="USER_LANGUAGE"/>
		<result property="userExcSymbol" column="USER_EXC_SYMBOL"/>
	</resultMap>
	
	
	<resultMap type="com.guivingAdmin.domain.Guiver" id="guiver">		
		<id property="guiverIdx" column="GUIVER_IDX"/>
		<result property="guiverFName" column="GUIVER_F_NAME"/>
		<result property="guiverLName" column="GUIVER_L_NAME"/>
		<result property="guiverBirth" column="GUIVER_BIRTH"/>
		<result property="guiverGender" column="GUIVER_GENDER"/>
		<result property="guiverEmail" column="GUIVER_PASSWORD"/>
		<result property="guiverPassword" column="GUIVER_EMAIL"/>
		<result property="guiverPhoneNum" column="GUIVER_PHONE_NUM"/>
		<result property="guiverAddr" column="GUIVER_ADDR"/>
		<result property="guiverCountry" column="GUIVER_COUNTRY_IDX"/>
		<result property="guiverLanguage" column="GUIVER_LANGUAGE"/>
		<result property="guiverFlag" column="GUIVER_FLAG"/>
		<result property="guiverStatus" column="GUIVER_STATUS"/>
		<result property="guiverType" column="GUIVER_TYPE"/>
		<result property="guiverUid" column="GUIVER_UID"/>
		<result property="guiverAddrCity" column="GUIVER_ADDR_CITY"/>
		<result property="guiverAddrState" column="GUIVER_ADDR_STATE"/>
		<result property="guiverAddrPostalCode" column="GUIVER_ADDR_POSTAL_CODE"/>
		<result property="guiverExcSymbol" column="GUIVER_EXC_SYMBOL"/>
		<collection property="guiverCompany" fetchType="lazy" column="GUIVER_IDX" select="com.guivingAdmin.dao.CompanyMapper.selectCompanyByGuiverIdx"/>
    	<collection property="guiverCitys" fetchType="lazy" column="GUIVER_IDX"  javaType="java.util.List" select="com.guivingAdmin.dao.CityMapper.selectCityByGuiverIdx"/>
	</resultMap>    
	
	<resultMap type="com.guivingAdmin.domain.Operator" id="operator">		
		<id property="opIdx"  column="OP_IDX"/>
		<result property="opFName"  column="OP_F_NAME"/>
		<result property="opLName"  column="OP_L_NAME"/>
		<result property="opPassword"  column="OP_PASSWORD"/>
		<result property="opPhoneNum"  column="OP_PHONE_NUM"/>
		<result property="opEmail"  column="OP_EMAIL"/>
		<result property="opAddr"  column="OP_ADDR"/>
		<result property="opCountry"  column="OP_COUNTRY_IDX"/>
		<result property="opLanguage"  column="OP_LANGUAGE"/>
		<result property="opStatus"  column="OP_STATUS"/>
		<result property="opFlag"  column="OP_FLAG"/>
		<result property="opUid"  column="OP_UID"/>
		<result property="opType"  column="OP_TYPE"/>
		<result property="opGender"  column="OP_GENDER"/>
		<result property="opPosition"  column="OP_POSITION"/>
		<result property="opBirth"  column="OP_BIRTH"/>
		<result property="opDepartment"  column="OP_DEPARTMENT"/>
		<result property="opExcSymbol"  column="OP_EXC_SYMBOL"/>
		<collection property="opCompany" fetchType="lazy" column="OP_IDX" select="com.guivingAdmin.dao.CompanyMapper.selectCompanyByOperatorIdx"/>
    	
	</resultMap>
	
	
	<resultMap type="com.guivingAdmin.domain.Vehicle" id="vehicle">	
		<id property="vehicleIdx"  column="VEHICLE_IDX"/>
		<result property="vehicleCarModelIdx"  column="VEHICLE_CAR_MODEL_IDX"/>
		<result property="vehicleModel"  column="VEHICLE_MODEL"/>
		<result property="vehicleNumber"  column="VEHICLE_NUMBER"/>
		<result property="vehicleYear"  column="VEHICLE_YEAR"/>
		<result property="vehicleColor"  column="VEHICLE_COLOR"/>
		<result property="vehicleOwnType"  column="VEHICLE_OWN_TYPE"/>
		<result property="vehicleOwnName"  column="VEHICLE_OWN_NAME"/>
		<result property="vehicleOwnImgUrl"  column="VEHICLE_OWN_IMG_URL"/>
		<result property="vehicleOwnAgreeImgUrl"  column="VEHICLE_OWN_AGREE_IMG_URL"/>
		<result property="vehicleRegedTime"  column="VEHICLE_REGED_TIME"/>
		<result property="vehicleStatus"  column="VEHICLE_STATUS"/>
		<result property="vehicleSeatCount"  column="VEHICLE_SEAT_COUNT"/>
		<result property="vehicleRegImgUrl"  column="VEHICLE_REG_IMG_URL"/>
		<collection property="vehicleGuiver" fetchType="lazy" column="VEHICLE_IDX" select="com.guivingAdmin.dao.GuiverMapper.selectGuiverByVehicleIdx"/>
		<collection property="vehicleCom" fetchType="lazy" column="VEHICLE_IDX" select="com.guivingAdmin.dao.CompanyMapper.selectCompanyByVehicleIdx"/>
	</resultMap>
	
	
	<resultMap type="com.guivingAdmin.domain.Reservation" id="reservation">
		<result property="resIdx" column="RES_IDX"/>
		<result property="resDate" column="RES_DATE"/>
		<result property="reservationStatus" column="RES_STATUS"/>
		<collection property="user" resultMap="user"/>
		<collection property="guiver" resultMap="guiver"/>
		<collection property="city" resultMap="city"/>
		<collection property="operator" resultMap="operator"/>
    	<collection property="company" resultMap="company"/>
    	<collection property="vehicle" resultMap="vehicle"/>
	</resultMap>
	

</mapper>
```
<br>
원래의 계획은 이러했다.<br>
<br>
1차적으로 로드되는 related Object는 outerJoin을 통해 로드한다.<br>
<br>
2차적으로 로드되는 related Object는 lazy load를 통해 로드한다.<br>
<br>
위 resultMap에 따르면 settlement가 가진 reservation은 1차적으로 로드되기 때문에 outer 조인을 사용하도록 하고<br>
<br>
reservation이 가진 guiver, operator 등의 related Object (city, company 등과 같은) 2차적인 로드는 lazy load를 사용한다는 것이다.<br>
<br>
(다시 살펴보니 아직 완벽하게 구현되진 않았다. 다시 수정해야할 듯)<br>
<br>
<br>
<br>
<br>
-----------------------------------------<br>
<br>
<br>
추가로 mybatis의 N+1과 같은 이슈 처리 방법을 찾아보다가 의문이 들어 글을 작성하게 되었다.<br>
<br>
https://www.facebook.com/groups/codingeverybody/permalink/3947287111978462/<br>
<br>
근데 답변을 아무도 안달아주네 ㅜㅜ<br>
<br>
<br>
위 resultMap을 통해 settlement 페이지는 리펙토링 적용이 끝났다.<br>
<br>
하지만 아직도 많이 남은 company , driver, operator 등등...<br>
<br>
모든 부분의 domain 비지니스 처리 , dto 통신, 페이징&검색필터링, view 및 공통 js 적용 등을 하면<br>
<br>
엄청난 노가다가 진행될 듯 하다.(이전의 노가다보단 훨씬 낫겠지 싶다.)<br>
<br>
<br>
