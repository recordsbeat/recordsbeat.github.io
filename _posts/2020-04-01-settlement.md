---
title: "본격 정산모듈작성3"
date: 2020-04-01 20:03:00 -0400
categories: mybatis paging
---

어제에 이어 오늘도 정산 페이지 작업

페이징과 검색필터 적용이 있겠다.

이 글을 쓰기까지 오지게 뚜까 맞고 왔다..

(페이징과 검색필터는 나만 어려운가 .. 다른 것들보다 훨씬 머리를 싸매고 만들었다.)

검색필터를 개발하며 나머지 페이지도 적용 될 수 있도록 염두하면서 개발을 하려 했으나..

생각대로 잘 되지 않아 일단 기능 구현에 먼저 초점을 두었다.

첫번째로 페이징

기존에 사용하던 pagingVO가 있다.





PagingVO 클래스

```

import lombok.Builder;
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;

@Getter
@Setter
@ToString
public class PagingVO {
	private int nowPage, startPage, endPage, total, cntPerPage, lastPage, start, end;
	private int cntPage = 10;
	
	public PagingVO() {
	}
	@Builder
	public PagingVO(int total, int nowPage, int cntPerPage) {
		setNowPage(nowPage);
		setCntPerPage(cntPerPage);
		setTotal(total);
		calcLastPage(getTotal(), getCntPerPage());
		calcStartEndPage(getNowPage(), cntPage);
		calcStartEnd(getNowPage(), getCntPerPage());
	}

// 제일 마지막 페이지 계산
	public void calcLastPage(int total, int cntPerPage) {
		setLastPage((int) Math.ceil((double) total / (double) cntPerPage));
	}

// 시작, 끝 페이지 계산
	public void calcStartEndPage(int nowPage, int cntPage) {
		setEndPage(((int) Math.ceil((double) nowPage / (double) cntPage)) * cntPage);
		if (getLastPage() < getEndPage()) {
			setEndPage(getLastPage());
		}
		setStartPage(getEndPage() - cntPage + 1);
		if (getStartPage() < 1) {
			setStartPage(1);
		}
	}

// DB 쿼리에서 사용할 start, end값 계산
	public void calcStartEnd(int nowPage, int cntPerPage) {
		setEnd(nowPage * cntPerPage);
		setStart(getEnd() - cntPerPage);
	}
}

```
인터넷 어디서 긁어온 vo인데 나름 잘 사용하는 중..


다음은 컨트롤러 부분 
```

	@RequestMapping({ "/list" })
	public ModelAndView listSettlement(
			@RequestParam(required=false,defaultValue="1") int nowPage,
			@RequestParam(required=false,defaultValue="10") int cntPerPage) {
		//default value를 사용하여 맨 처음 페이지 진입값 세팅
    
		ModelAndView mav = new ModelAndView("settlement/list.tiles");
		try {	
			int total = settlementService.selectAllCount();
      //builder .. 잘 써먹고 있다.
			PagingVO paging = PagingVO.builder()
					.nowPage(nowPage)
					.cntPerPage(cntPerPage)
					.total(total)
					.build();
			
			
			List<EnumValue> settleStatus = ValueUtils.getMapper().getAll().get("SettleStatus");
			//paging 파라미터 추가
			List<SettleListResponseDto> resList = settlementService.selectAllSettlement(paging);
			
      //paging 객체를 다시 클라이언트로 전달하려는 목적. vo를 map으로 바꿔주는 메소드..
			mav.addObject("paging", CommonUtils.convertObjectToMap(paging));
			mav.addObject("settleStatus", settleStatus);
			mav.addObject("result", resList);
			
		} catch (Exception e) {
			this.logger.error("ERROR - ", e);
		}
		return mav;
	}	


	@RequestMapping({ "/search" })
	public ModelAndView searchSettlement(SettlementListSearchVO search,
			@RequestParam(required=false,defaultValue="1") int nowPage,
			@RequestParam(required=false,defaultValue="10") int cntPerPage) {
		ModelAndView mav = new ModelAndView("settlement/list.tiles");
		try {	
			
			int total = settlementService.selectAllCount(search);
			PagingVO paging = PagingVO.builder()
					.nowPage(nowPage)
					.cntPerPage(cntPerPage)
					.total(total)
					.build();
			
			List<EnumValue> settleStatus = ValueUtils.getMapper().getAll().get("SettleStatus");

			List<SettleListResponseDto> resList = settlementService.selectAllSettlement(search,paging);
			
			mav.addObject("paging", CommonUtils.convertObjectToMap(paging));
			mav.addObject("search", CommonUtils.convertObjectToMap(search));	
			mav.addObject("settleStatus", settleStatus);
			mav.addObject("result", resList);
		} catch (Exception e) {
			this.logger.error("ERROR - ", e);
		}
		return mav;
	}	


```
컨트롤러 /list 접근 시 현재 페이지와 페이지당 노출 row의 갯 수를 받아온다. ( 없으면 디폴트 세팅 )

paging vo객체를 만들어 쿼리까지 전달 끝

/seach 부분은 SettlementListSearchVO 객체 매핑해서 받아오기(이것도 익숙치 않아서 의외로 시간 많이 걸렸다..)

사실 /search 매핑을 따로 사용하지 않고 /list 하나로 사용하고 싶었으나 SettlementListSearchVO 를 가져오는데 애 먹어서 따로 빼뒀다.

(이전에는 hashmap으로 좀 지저분하게 하나로 처리했다는)







인터넷에서 오려온 convertObjectToMap
```
public static Map convertObjectToMap(Object obj) {
  Map map = new HashMap();
  Field[] fields = obj.getClass().getDeclaredFields();
  for (int i = 0; i < fields.length; i++) {
    fields[i].setAccessible(true);
    try {
      map.put(fields[i].getName(), fields[i].get(obj));
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
  return map;
}
```
잘썼어요 고맙습니다..







서비스 부분

```
@Transactional(readOnly = true)
	public List<SettleListResponseDto> selectAllSettlement(PagingVO paging) throws Exception{
		return settlementMapper.selectAllSettlement(paging)
				.stream()
				.map(SettleListResponseDto::new)
				.collect(Collectors.toList());
	}
	@Transactional(readOnly = true)
	public List<SettleListResponseDto> selectAllSettlement(SettlementListSearchVO search,PagingVO paging) throws Exception{
		return settlementMapper.selectSearchSettlement(search,paging)
				.stream()
				.map(SettleListResponseDto::new)
				.collect(Collectors.toList());
	}	
	@Transactional(readOnly = true)
	public int selectAllCount() throws Exception{
		return settlementMapper.selectAllCount();
	}
	@Transactional(readOnly = true)
	public int selectAllCount(SettlementListSearchVO search) throws Exception{
		return settlementMapper.selectAllCount(search);
	}
```
나름 오버로딩 해본 건데 역시나 지저분하다.. 페이징을 위한 selectAllCount 메소드가 오버로드 되었고

selectAllSettlement는 그냥 다른 메소드 호출한다. (쿼리 때문에 나눠버림 ㅠㅠ)


그리고 급 눈에 띄인 Transactional 어노테이션

참고링크를 통해 공부해봐야겠다.


https://goddaehee.tistory.com/167







DAO 부분 

오늘 처음 안거 있다.

```
List<Reservation> selectAllSettlement(PagingVO paging) throws Exception;
List<Reservation> selectSearchSettlement(@Param("search")SettlementListSearchVO search,@Param("paging")PagingVO paging) throws Exception;
int selectAllCount() throws Exception;
int selectAllCount(SettlementListSearchVO search) throws Exception;
```
@Param 어노테이션 쓰면 hashmap 처럼 쓸 수 있다. 이거 좋은 듯 

근데 mybatis 사용법 미숙으로 이 마저 또 헤맸다.








sql 문

```
<select id="selectAllSettlement" resultMap="com.guivingAdmin.dao.ResultMap.reservationResult">
		SELECT 
			RD.RES_IDX,...
			ST.STATUS AS SETTLE_STATUS, ST.TR_TIME AS SETTLE_TIME
		FROM VW_RESERVATION_DETAIL RD,
		    (SELECT ... FROM
          TB_SETTLEMENT ST,
          (SELECT MAX(ST.SETTLEMENT_IDX) AS MAX_IDX
          FROM TB_SETTLEMENT ST
          GROUP BY ST.RES_IDX) SUB
		    WHERE ST.SETTLEMENT_IDX = SUB.MAX_IDX)ST    
		WHERE ...
		ORDER BY RES_DATE DESC, RD.RES_IDX DESC
		LIMIT #{start},#{end};
	</select>
  
  <select id="selectSearchSettlement" resultMap="com.guivingAdmin.dao.ResultMap.reservationResult">
		SELECT 
			(...)
		FROM VW_RESERVATION_DETAIL RD,
		    (...)  
		WHERE RD.RES_IDX = ST.RES_IDX
		<if test="search.status != null">AND STATUS = #{search.status}</if>
		<if test="search.fromDate != null and search.toDate != null">
			AND RES_DATE BETWEEN DATE_FORMAT('${search.fromDate}','%Y-%m-%d') AND DATE_FORMAT('${search.toDate}','%Y-%m-%d')
		</if>
    	<if test="search.searchWord != null">AND ${search.searchOpt} LIKE '%${search.searchWord}%'</if>
		ORDER BY RES_DATE DESC, RD.RES_IDX DESC
		LIMIT #{paging.start},#{paging.end};
	</select>
  
  <select id="selectAllCount" resultType="int">
		SELECT 
			COUNT(RD.RES_IDX)
		FROM VW_RESERVATION_DETAIL RD,
		   (...)
		WHERE RD.RES_IDX = ST.RES_IDX
		<if test="status != null">AND STATUS = #{status}</if>
		<if test="fromDate != null and toDate != null">
			...
		</if>
    	<if test="searchWord != null">.../if>
		ORDER BY ...;
	</select>
```

selectAllSettlement에서 바뀐거라곤 #{start},#{end} limit 뿐... 

한가지 더하자면 from 쪽 서브쿼리가 존재하는 이유는 데이터를 update하는 것이 아닌 insert 후에 최상위 건을 불러오는 방식을 쓰기 위하여
(history 개념을 갖고 싶었다..)

진짜 뻘 짓하느라 시간 뺏긴 부분은 selectAllCount 부분에 있다.

AND STATUS = #{status} 이 부분에서 ${status} 로 잘못 사용해 두 시간 정도 뻘 짓

(내가 알기론 ${...}는 데이터 자체 출력 #{...} 파라미터로 변환된 값 출력)

이전에 mybatis TypeHandler를 통해 enum을 db와 통신할 때는 숫자 코드로 컨버팅 해주도록 하였다.

그러나 ${...}를 쓰면 컨버팅 된 값이 아닌 enum getName()의 그것이 찍혀버리는 낭패.. 모쪼록 담부터 조심하는걸로










페이지 부분

```
<!-- 페이징  -->
  <div class="pagination">

    <c:if test="${paging.startPage != 1 }">
      <a href="" class="direction prev page" data-no="${page.nowPage()-1}">이전</a>
    </c:if>

    <c:forEach begin="${paging.startPage}" end="${paging.endPage}" var="p">
      <c:choose>
        <c:when test="${p == paging.nowPage }">
          <a href="" class="page active" data-no="${p}" >${p}</a>
        </c:when>
        <c:when test="${p != paging.nowPage }">
          <a href="" class="page" data-no="${p}" >${p}</a>
        </c:when>
      </c:choose>
    </c:forEach>

    <c:if test="${paging.endPage != paging.lastPage}">
      <a href="" class="direction next page" data-no="${page.nowPage()+1}">다음</a>
    </c:if>

			
  </div>
```
사실 페이징도 별거 없긴 한데... class 추가해서 클릭 이벤트 넣고 data-no 에서 해당 페이지 번호 넘겨주는 거다.







javascript
```

$('.page').click(function(){
  _this.gopage($(this).data('no'));
});
(...)

loadpage : function(){
			var _this = this;
			_this.setparams();
			$("#searchForm").submit();
		},
gopage : function(pageNo){
  $("#nowPage").val(pageNo);
  var _this = this;
  _this.loadpage();
},
```
요정도..

검색필터를 적용한 상태에서 페이지 이동을 하더라도 검색필터가 남아있는 것을 의도하였다.

문제일수도 있는 건 검색필터를 변경하고 search 버튼이 아닌 페이지 번호를 누르면 검색필터가 적용된다는 점...







다 만들고 보면 별 거 아닌듯한데 항상 만들 때마다 애먹는 편이다.

최대한 중복 코드 줄이고 재사용하고 싶었으나 코딩 방식이 잘못된 탓인지 중복이 발생해 버렸다.

(컨트롤러의 /list 와 /search 내에 중복이 있는 점, mybatis내에 count 쿼리와 search 쿼리의 where 절이 중복인 점..)




데이터가 많지않아 테스트를 제대로 해보진 못했으나


이도 내일 마저 하는걸로.

왠지 진빠지는 하루



