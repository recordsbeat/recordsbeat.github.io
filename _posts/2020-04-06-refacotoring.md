---
title: "생각대로 되지 않는 군"
date: 2020-04-06 01:16:00 -0400
categories: mybatis collection oneToMany RestApi
---

<br>
오늘은 두가지 작업을 진행.<br>
<br>
새벽에 생각했던 비지니스 로직은 도메인에서, api는 행위에 따라서 디자인하기<br>
<br>
그리고 mybatis one to many select (one to one 포함) 매핑 진행<br>
<br>
<br>
<br>
<br>
첫번째부분에서<br>
<br>
어제 참조했던 링크<br>
<br>
https://www.popit.kr/%EC%A2%8C%EC%B6%A9%EC%9A%B0%EB%8F%8C-api-%ED%95%A8%EC%88%98-%EC%9D%B4%EB%A6%84%EC%9D%80-%EC%96%B4%EB%96%BB%EA%B2%8C-%EC%A7%93%EB%8A%94%EA%B2%8C-%EC%A2%8B%EC%9D%84%EA%B9%8C/?fbclid=IwAR3eRMiNa4ncY-X8WmqrXOQtAq8dxvqyNo_BfrneJnFGPrifLnGYgk3JfR4<br>
<br>
<br>
이부분과 rest api의 디자인 개념과는 사뭇 달랐다.<br>
<br>
위의 링크를 따라 사용자가 status를 제공하는 것이 아닌 행위 목적에 맞는 값을 서버에서 update 할 것으로 받아들였고,<br>
<br>
이를 실행에 옮기려 했으나..<br>
<br>
rest api 가로되, <br>
<br>
동사로 행위를 나타내지 않는다. URI를 통해 resource 를 나타내고 <br>
<br>
header의 메소드형태 (get,post,put,delete)에 따라 행위를 진행한다.<br>
<br>
<br>
이 두가지 개념이 충돌한다. <br>
<br>
행위로 api를 개발하면 수많은 분기문을 없애고 하나의 로직에 집중할 수 있다.<br>
vs<br>
rest api의 URI는 메소드와 리소스로서 행위를 판단한다.<br>
<br>
무엇이 맞을까? 라며 위 링크를 계속 보다가<br>
<br>
글쓰기 직전에 이 단어가 보인다.<br>
<br>
'API 함수 이름' URI가 아닌 함수 이름인건가..? <br>
<br>
일단 해당부분에 대한 판단은 좀 더 고민 후에 내려야지 싶다.<br>
<br>
---------------------------------------------------------<br>
<br>
<br>
rest api에 관하여 공부를 하며 이동욱님의 책을 다시 보니 <br>
<br>
RestController 와 Controller가 분리되어 있었다.<br>
<br>
하여 또 한 번 찾아보았다.<br>
<br>
링크<br>
<br>
https://mangkyu.tistory.com/49'<br>
<br>
<br>
내가 내린 결론은 다음과 같다.<br>
<br>
rest api는 다양화된 플랫폼이 하나의 인프라와 통신하기 위한 서비스. (확장성을 염두하고 만든다는 느낌)<br>
<br>
그래서 stateless라는 말이 나왔나 싶다.<br>
<br>
아무튼 그래서 내가 한 것은 restcontroller 와 controller 분리<br>
<br>
<br>
<br>
컨트롤러<br>

```

@Controller
@RequiredArgsConstructor
public class IndexController {
	protected Logger logger = LoggerFactory.getLogger(getClass());
	
	private final SettlementService settlementService;
	
	@RequestMapping(value="/settlement" , method = {RequestMethod.GET, RequestMethod.POST})
	public ModelAndView listSettlement(SettlementListSearchDto search,
			@RequestParam(required=false,defaultValue="1") int nowPage,
			@RequestParam(required=false,defaultValue="10") int cntPerPage) {
		
		ModelAndView mav = new ModelAndView("settlement/list.tiles");
		try {	
			
			int total = settlementService.selectAllCount(search);
			PagingDto paging = PagingDto.builder()
					.nowPage(nowPage)
					.cntPerPage(cntPerPage)
					.total(total)
					.build();
			
			
			List<EnumValue> settleStatus = ValueUtils.getMapper().getAll().get("SettleStatus");
			
			List<SettlementListResponseDto> resList = settlementService.selectAllSettlement(search,paging);

			mav.addObject("paging", CommonUtils.convertObjectToMap(paging));
			mav.addObject("search", CommonUtils.convertObjectToMap(search));	
			mav.addObject("settleStatus", settleStatus);
			mav.addObject("result", resList);
			
		} catch (Exception e) {
			this.logger.error("ERROR - ", e);
		}
		return mav;
	}
}

```
<br>
<br>
검색과 조회는 GET매핑으로 진행된다고 rest api에서 이야기한다.<br>
<br>
내가 일전에 추가해두었던 검색 필터링은 post방식에서 작동하도록 되어있다.<br>
<br>
그래서 최초에는 GET 매핑 검색 필터링을 위해서는 post매핑을 받아들이도록 설정하였다.<br>
<br>
(사실 form submit method를 get방식으로 처리해도 되긴 하지만 url이 너무 지저분해져서 post를 유지하기로 결정)<br>
<br>
<br>
restcontroller
```

@RequiredArgsConstructor
@RestController
public class SettlementController {
	protected Logger logger = LoggerFactory.getLogger(getClass());
	
	private final SettlementService settlementService;
	
	@PutMapping("/api/v1/settlement")
	@ResponseBody
	public int putSettlement(@RequestBody List<SettlementSaveRequestDto> resList) throws Exception {
		int result = 0;
		try {
			result = this.settlementService.putSettlement(resList);
		} catch (Exception e) {
			this.logger.error("ERROR - ", e.getMessage());
			e.printStackTrace();
		}
		return result;
	}
}
```
<br>
<br>
Put매핑을 통해 해당 기능을 명시하였다.(사실 특정row update가 아닌 insert 방식이기 때문에 Post가 맞을지도 모르겠다.)<br>
<br>
책을 따라서 api버전을 달아두었고 그 외 특별한 것은 없다.(메소드 명도 바꿈)<br>
<br>
<br>
목표한 바는 /api/v1/standby ,  /api/v1/complete ,  /api/v1/cancel 과 같은 행위에 따른 분류였다.<br>
<br>
하지만 위에서 이야기한 것처럼 이 부분에 대한 판단은 좀 더 미루어 두도록하자..<br>
<br>
<br>
<br>
--------------------------------------------------------------------
<br>
<br>
<br>
두번째는 domain에 대한 나의 개념이 잘못되었던 것.<br>
<br>
domain 혹은 entity로 대조될 수 있는 객체를 쿼리를 통해 임의로 작성하였다.<br>
<br>
(맞는건지 틀린 건지는 아직 잘 모르겠으나 목적은 entity처럼 사용하는 것이기 때문에)<br>
<br>
하여 domain 클래스를 재정의하고 모든 데이터를 불러와 resultMap에 매칭 시키는 것이아닌<br>
<br>
하나의 domain객체 그리고 다른 domain 객체를 필드로 갖는 것으로 하였다.<br>
<br>
다음과 같은 mybatis 재설정<br>
<br>
<br>

```
<mapper namespace="ResultMap">

	<resultMap type="com.guivingAdmin.domain.City" id="city">
		<result property="city" column="CITY"/>
		<result property="country" column="COUNTRY"/>			
		<result property="currency" column="CURRENCY"/>
	</resultMap>
	<resultMap type="com.guivingAdmin.domain.Settlement" id="settlement">
		<result property="resIdx" column="RES_IDX"/>
		<result property="status" column="SETTLE_STATUS"/>			
		<result property="trTime" column="SETTLE_TIME"/>
	</resultMap>
  
	<resultMap type="com.guivingAdmin.domain.Reservation" id="reservation">
		<id property="resIdx" column="RES_IDX"/>
    	<collection property="city" column="CITY_IDX" select="com.guivingAdmin.dao.CityMapper.selectCityByCityIdx"/>
    	<collection property="settlement" column="RES_IDX" select="com.guivingAdmin.dao.SettlementMapper.selectSettlementByResIdx"/>
	</resultMap>
</mapper>

```
<br>
<br>
```
<select id="selectSettlementByResIdx" resultMap="ResultMap.settlement">
		SELECT 
		    ST.RES_IDX,
				ST.STATUS AS SETTLE_STATUS, ST.TR_TIME AS SETTLE_TIME
			FROM (SELECT ST.RES_IDX, ST.STATUS, ST.TR_TIME FROM
			    TB_SETTLEMENT ST,
			    (SELECT MAX(ST.SETTLEMENT_IDX) AS MAX_IDX
			    FROM TB_SETTLEMENT ST
			    GROUP BY ST.RES_IDX) SUB
			    WHERE ST.SETTLEMENT_IDX = SUB.MAX_IDX)ST    
			WHERE ST.RES_IDX = #{resIdx}
	</select>
  
  <select id="selectCityByCityIdx" resultMap="ResultMap.city" >
		SELECT CITY_IDX AS CITY,CITY_CURRENCY AS CURRENCY,
			COUNTRY_IDX AS COUNTRY
			FROM TB_CITY
		WHERE CITY_IDX = #{cityIdx}
	</select>
```
<br>
<br>
오늘은 대단하진 않지만 테스트 코드로 돌려봤다.<br>
<br>
레알 편하긴하다<br>
<br>
```

@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class SettlementServiceTest {
	
	@Autowired
	SettlementMapper sm;
	
	@Test
	public void testHomeInit() throws Exception {
		SettlementListSearchDto search = SettlementListSearchDto.builder()
				.build();
		PagingDto paging = PagingDto.builder().nowPage(1)
				.cntPerPage(10)
				.total(10)
				.build();
		List<Reservation> rList = sm.testSelect(search,paging);
		System.out.println("listlistlist : " + rList);
	}

}

```
<br>
<br>
덕분에 매핑 잘되는지 빠르게 확인했습니다...<br>
<br>
<br>
<br>
마지막으로 lazy loading에 대한 생각.<br>
<br>
위는 사실 one to many가 아닌 one to one 이기 때문에 크게 부하가 걸리지 않아보인다.(데이터도 별로없고)<br>
<br>
하지만 나중에 늘어날 데이터와 list형태등의 one to many상황에서는 lazy loading (객체에 대해 get 요청할 때 select 쿼리 실행)<br>
<br>
할 수 있도록 설정을 찾아봐야겠다.<br>
<br>
<br>
<br>
<br>
오늘은 작업한 것보다 찾아본 양이 더 많다.<br>
<br>
내일 마저 collection을 사용한 resultMap 작성 뒤 매핑 테스트,<br>
<br>
domain 에서 비지니스 로직 처리 등이 과제로 남아있다.
