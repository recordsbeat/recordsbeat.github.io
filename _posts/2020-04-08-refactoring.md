---
title: "도메인 모델이란 뭘까"
date: 2020-04-08 18:53:00 -0400
categories: domainmodel domain entity
---
<br>
<br>
어제 이야기한 N+1의 전략은 이러하다.<br>
<br>
1. 1차적으로 필요한 연관객체는 outer join을 사용해(mysql view로 미리 선언) resultMap에 매치되는 칼럼데이터를 뽑아낸다.<br>
<br>
2. 연관 객체 내에 연관 객체 (1:N:1 , 1:N:N 등..)는 lazy load를 사용하여 가져온다.<br>
<br>
역시 쳐맞기전까지 계획은 나름 있었지만 테스트 코드를 돌리고 나니 1차 연관 객체를 get했더니 2차 3차 연관 객체까지 나와버렸다.<br>
(N+M 상황)<br>
이는 결국 각각 연관객체에 필요한 부분까지만 명시를 해두어야 했고 아래와 같이 extend를 사용해 완벽히 로드되어야할(entity성격)<br>
resultMap과 연관객체로 사용될 resultMap을 분리하였다.<br>





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
	</resultMap>
	
	<resultMap type="com.guivingAdmin.domain.Company" id="companyExt" extends="company">	
		<result property="opCount" column="OP_COUNT"/>	
		<result property="guiverCount" column="GUIVER_COUNT"/>	
		<result property="vehicleCount" column="VEHICLE_COUNT"/>	
		<collection property="comAddrState" column="VEHICLE_COUNT"/>	
    	<collection property="citys"  fetchType="lazy" column="COM_IDX" javaType="java.util.List" select="com.guivingAdmin.dao.CityMapper.selectCityByComIdx"/>		 
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
	</resultMap>    
	
	<resultMap type="com.guivingAdmin.domain.Guiver" id="guiverExt" extends="guiver">		
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
	</resultMap>
	
	<resultMap type="com.guivingAdmin.domain.Operator" id="operatorExt" extends="operator">		
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
	</resultMap>
	
	<resultMap type="com.guivingAdmin.domain.Vehicle" id="vehicleExt" extends="vehicle">	
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


그리하야 열심히 짜둔 outer 쿼리를 사용하여 이모저모 연관객체를 모아올 수 있었다.<br>
<br>
<br>
<br>
<br>
<br>
<br>
이를 토대로 페이지 리펙토링 작업에 들어가야 했다.<br>
<br>
첫번째는 업체 페이지 !<br>
<br>
조회 페이지를 보다가 ... ???<br>
예상치 못한 일 entity만 가지고 가지고 올 수 없는 데이터가 있었다. (각 업체의 차량 갯수 / 직원 수 같은)<br>
이를 위해서 entity를 뜯어고치는 건 말도 안되는 일 . 쿼리모양 따라서 entity가 변하는건 entity가 아닌거다.<br>
그렇다고 쿼리마다 entity와 새로운 객체를 만들어서 써야할까?<br>
<br>
내 생각이 도달했던 지점은 이러했다.<br>
<br>
1. 조회(통계 수치)와 같이 화면에 따라 변하는 데이터는 hashmap으로 통신한다.<br>
이미 이렇게 사용하고 있었다. 이 방식을 쓰면 column명 들이 카멜케이스 변환이 아닌 column명 그대로 hashmap에 담기게 된다.<br>
그리고 일일히 칼럼명과 view단을 매칭시켜야되는 수고스러움 발생<br>
(사실 이제껏 이렇게 개발했다.)<br>
<br>
2. hashmap에 entity객체와 부가적인 데이터를 담아서 통신한다.<br>
이렇게 할 뻔 했다. 하지만 생각해보니 resultMap을 다시 정의해주고 , entity를 다시 dto로 변환해서 hashmap에 넣고..<br>
view에서 데이터를 가져올 땐 뎁스가 너무 많이 발생할 것 같았다. ex)result.dto.city.getValue()<br>
<br>
3. 그냥 dto 새로 만들기 <br>
결과적으로 말하자면 list조회를 위한 dto를 새로 만들었다.<br>
만들다보니 결국 '이럴거면 쿼리문마다 필요한 dto를 만들었던 것과 다르지 않잖아?'<br>
맨처음 스프링을 사용했을 때 쿼리문에 따라서 dto를 계속해 만들어야 했고 이는 쿼리가 만들어질 때 마다 계속해<br>
dto클래스가 정의되었어야 했다.<br>
<br>
참고링크<br>
https://okky.kr/article/335497<br>
<br>
'entity에 포함되지 않는 데이터는 어떻게 뽑아 써야합니까?'<br>
<br>
그리고<br>
<br>
참고링크<br>
https://ict-nroo.tistory.com/117<br>
<br>
'JPA 실무 경험 공유'<br>
두 가지 링크의 공통점은 조회를 위한 데이터는 어떤 방식으로 뽑아와야하는가?<br>
DAO 레이어단의 entity<->dto 통신 방식에서 조회나 통계를 위한 자료는 entity내에 포함되지 않는 경우가 많다.<br>
<br>
첫번째 링크 제타건담님의 말<br>
<br>
'애초에 개발하신 분이 JPA와 현실을 접목시킬때..<br>
너무 원론적으로 치중해서 개발한 느낌이 드네요..<br>
예를 들어 댓글 count 같은 경우도 이걸 개개당 사용자가 요청할때 조회해야 하는 것인지..<br>
아니면 첨부터 다 같이 조회해야 하는 것인지..그런거에 대한 고민이 전혀 없었어요..<br>
혹시 엔티티에 대한 접근을 매번 엔티티 그래프 개념으로 접근하는건 아닌지 모르겠군요..'<br>
<br>
- flexible 하게 왔다갔다하면서 써라 ! 라는 뜻 맞나? .. 엔티티와 엔티티그래프의 차이가 무엇인지는 찾아봐야겠다.<br>
<br>
두번째 링크<br>
<br>
'통계 쿼리처럼 매우 복잡한 SQL은 어떻게 하나요?<br>
-거의 다 QueryDSL로 처리하고, DTO로 뽑아낸다.<br>
정말 안될 경우 네이티브 쿼리 사용한다.'<br>
<br>
이 말을 보고 결정적으로 DTO로 가져가게 되었다.<br>
<br>
entity의 연관 객체로써 매핑되어야할 쿼리가 아니라면 굳이 entity에 묶여있을 이유가 없던 것이다.<br>
<br>
<br>
실행에 옮겼습니다.<br>



```
<sql id="sqlWhere">
		<include refid="Function.checkEmpty"/>
		<if test="#fn=isNotEmpty, #fn(search.city)">
			AND CT.CITY_IDX = #{search.city}
		</if>
		<if test="#fn=isNotEmpty, #fn(search.searchWord)">
    		AND ${search.searchOpt} LIKE '%${search.searchWord}%'
    	</if>
	</sql>

	<select id="selectCompanyList" resultType="com.guivingAdmin.web.dto.CompanyListResponseDto">
		SELECT
			CP.COM_IDX, CP.COM_NAME, CP.COM_AUTH_CODE,CP.COM_BIZ_NUM,
			CP.COM_OWNER_NAME, CT.CITY_IDX AS CITY, CT.COUNTRY_IDX AS COUNTRY,
			IFNULL(OP_SUB.OP_COUNT,0) AS OP_COUNT,
	      	IFNULL(GUIVER_SUB.GUIVER_COUNT,0) AS GUIVER_COUNT,
	      	IFNULL(VEHICLE_SUB.VEHICLE_COUNT,0) AS VEHICLE_COUNT,
			DATE_FORMAT(CP.COM_BUILD_DATE,'%Y-%m-%d') AS COM_BUILD_DATE
		FROM 
		TB_COMPANY CP
		LEFT JOIN 
			(SELECT COUNT(OP.OP_IDX) AS OP_COUNT, OP.OP_COMPANY_IDX AS COM_IDX FROM TB_OPERATOR OP GROUP BY OP.OP_COMPANY_IDX)OP_SUB
			ON CP.COM_IDX =OP_SUB.COM_IDX
		LEFT JOIN 
			(SELECT COUNT(GV.GUIVER_IDX) AS GUIVER_COUNT, GV.GUIVER_COMPANY_IDX AS COM_IDX FROM TB_GUIVER GV GROUP BY GV.GUIVER_COMPANY_IDX)GUIVER_SUB
			 ON CP.COM_IDX =GUIVER_SUB.COM_IDX
		LEFT JOIN 
			(SELECT COUNT(VH.VEHICLE_IDX) AS VEHICLE_COUNT, VH.VEHICLE_COM_IDX AS COM_IDX FROM TB_VEHICLE VH GROUP BY VH.VEHICLE_COM_IDX)VEHICLE_SUB
			ON CP.COM_IDX =VEHICLE_SUB.COM_IDX
		, TB_COM_CITY CC, TB_CITY CT
		WHERE CP.COM_IDX = CC.COM_IDX
		AND CC.CITY_IDX = CT.CITY_IDX
		<include refid="sqlWhere"/>
		ORDER BY CP.COM_IDX DESC
		<include refid="Function.pagingLimit"/>
	</select>
```

```

@Getter
@ToString
public class CompanyListResponseDto {
	private Long comIdx;
	private String comName;
	private String comAuthCode;
	private String comOwnerName;
	private CountryEnum country;
	private CityEnum city;
	private int opCount;
	private int guiverCount;
	private int vehicleCount;
	private String comBuildDate;

}

```
<br>
나머지 코드는 이전 정산과 크게 다를 것이 없기에 패스<br>
CRUD중 R와 같은 행위에서 굳이 entity를 따라갈 필요가 없다는 것<br>
나머지 CUD는 연관객체들과의 일관성을 위해 entity로 함께 진행되면 될 것 같다는 어렴풋한 이해를 했다.<br>
<br>
하지만 실제 비지니스가 진행되는 것과 CRUD를 실행하는 것과의 괴리는 좁힐 수가 없었다.<br>
하여 다시 한 번 이 개발방식의 목적을 생각해봤다.<br>
<br>
<br>
mybatis 를 ORM처럼 사용하여 연관 객체를 불러오고 테이블과 매핑되는 entity 객체만든다.<br>
그리고 그 안에서 비지니스를 처리한다.<br>
<br>
<br>
처음 생각은 이랬다. 그리고 당연하다는 듯이 테이블과 매칭되는 클래스를 '도메인'으로 정의했다.<br>
table = entity = domain 이라 생각했던 것, 완전히 틀렸던거다.<br>
<br>
<br>
일단 도메인은 어떠한 서비스나 행위가 소프트웨어적인 범주로 정의된 것 정도로 이해했다.<br>
<br>
그 도메인안에 객체(행위주체)와 행위가 나누어져서 도메인 모델 방식의 개발이 되는 것이다..<br>
<br>
참고링크<br>
<br>
https://jojoldu.tistory.com/346<br>
<br>
https://velog.io/@hyunssooo/2019-08-19-0008-%EC%9E%91%EC%84%B1%EB%90%A8<br>
<br>
https://woowabros.github.io/study/2016/07/07/think_object_oriented.html<br>
<br>
뭔가 더 쓰려고 했는데 대부분의 설명이 저기에 들어있네..<br>
<br>
일단 새로 알게 된 것 VO를 DTO랑 혼용되어 쓰는 경우가 있었고 이 것은 통상적으로 잘못되었다. 까진 알았지만<br>
VO를 좀 더 보자면 식별자를 띄지 않는 데이터 필드 . 데이터 덩어리라고 보면 되겠다.<br>
가령 가격를 int price 표현하기보단 class price{ int value;} 가 더 좋지 않을까?<br>
<br>
위의 글들을 읽고 지금 다루는 프로젝트에 대입해보려고 했으나 크게 감이 오지 않았다.<br>
<br>
한가지 차이점으로 보이는 것은 <br>
DB(테이블)을 먼저 잡고 Entity를 구성하느냐<br>
Entity를 먼저 잡고 DB를 구성하느냐의 차이가 보인다.<br>
필요한 데이터를 도메인에서 entity별로 정의한다면 데이터들의 사용이 명확해지고 CRUD의 제약을 크게 받지 않을 것이라는 생각이 들었다.<br>
지금의 프로젝트는 mybatis를 사용하며 트랜잭션 스크립트내지 절차지향적인 개발을 취하고 있었기 때문에<br>
도메인 모델 패턴의 방식을 적용하는 것이 까다로워 보인다.<br>
<br>
(사실 비지니스 적으로 녹여낼 것이 많이 없다 라는 판단도 든다. 당장은 어드민 서비스의 CRUD만을 만들고 있기 때문에 <br>
위에서 본 글들 처럼 어떤 포인트를 녹여내야할지 감이 안온다.)<br>
<br>
<br>
마지막으로 엉겁결에 java8테스트 코드를 작성했던 부분<br>
<br>


```
	@Test
	@Ignore
	public void testFunc() {
		
		Arrays.asList(CityEnum.values())
		.stream()
		.map(CodeEnum.class::cast)
		.forEach(x-> System.out.println("sosodfv" + x.getComment()));
		
		
		CityEnum[] en = CityEnum.values();
		
		List<? extends CodeEnum> tlist = (List)Arrays.asList(en);
		tlist.stream().forEach(System.out::println);
		
	}
```


쓰다보니 알 것 같고 쓰다 보니 편하다. <br>


