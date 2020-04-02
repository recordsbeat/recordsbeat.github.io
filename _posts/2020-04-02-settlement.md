---
title: "본격 정산모듈작성4"
date: 2020-04-02 19:15:00 -0400
categories: mybatis paging js
---

정산 페이지의 페이징과 검색을 마지막으로 작업하였다. (앞으로 더 하겠지만)

어제 대략적으로 정리된 코드들을 다시 봤는데 처참할 정도로 더러웠다.

다시 작성 중에 어제까지 안되었던 /list컨트롤러만으로 search까지 구현하기가 오늘 갑자기(?) 되어버렸다.

그나마 깔끔해진거 같은 코드 다행인듯..



검색 dto
```

@Getter
@RequiredArgsConstructor
@ToString
public class SettlementListSearchDto{
	private final String searchOpt;
	private final String searchWord;
	private final String fromDate;
	private final String toDate;
	private final SettleStatus status;
	
	
}
```

컨트롤러
```
@RequestMapping({ "/list" })
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
```

검색 dto를 받아서 null 이면 검색 조건에 포함x 값이 있으면 필터링 하는 구조로 바꿔봤다.

어제해놨던 오버로딩은 더 이상 필요없어서 지워버림!


서비스
```

	@Transactional(readOnly = true)
	public List<SettlementListResponseDto> selectAllSettlement(SettlementListSearchDto search,PagingDto paging) throws Exception{
		return settlementMapper.selectAllSettlement(search,paging)
				.stream()
				.map(SettlementListResponseDto::new)
				.collect(Collectors.toList());
	}	
	@Transactional(readOnly = true)
	public int selectAllCount(SettlementListSearchDto search) throws Exception{
		return settlementMapper.selectAllCount(search);
	}
```


dao의 selectAllCount @param 어노테이션을 사용, count 쿼리와 list 쿼리가 동일한 where조건문을 쓰도록 하였다.


(이전에는 필드의 depth가 달랐다..)

```

@Repository
public interface SettlementMapper {	
	List<Reservation> selectAllSettlement(@Param("search")SettlementListSearchDto search,@Param("paging")PagingDto paging) throws Exception;
	int selectAllCount(@Param("search")SettlementListSearchDto search) throws Exception;
	int updateSettlementStatus(List<Reservation> domainList) throws Exception;
}

```


mybatis 부분에서 좀 공들인 부분이 생겼다.

java의 메소드를 콜하는 부분이 있던 것.

search dto의 null 혹은 공백 체크하기에 안성맞춤이었다.


참고링크
https://hayoung950518.tistory.com/43

https://cofs.tistory.com/309


공백 체크 메소드 작성
```
/**
     * Object type 변수가 비어있는지 체크
     * 
     * @param obj 
     * @return Boolean : true / false
     */
    public static Boolean empty(Object obj) {
        if (obj instanceof String) return obj == null || "".equals(obj.toString().trim());
        else if (obj instanceof List) return obj == null || ((List<?>) obj).isEmpty();
        else if (obj instanceof Map) return obj == null || ((Map<?, ?>) obj).isEmpty();
        else if (obj instanceof Object[]) return obj == null || Array.getLength(obj) == 0;
        else return obj == null;
    }
 
    /**
     * Object type 변수가 비어있지 않은지 체크
     * 
     * @param obj
     * @return Boolean : true / false
     */
    public static Boolean notEmpty(Object obj) {
        return !empty(obj);
    }

```


mybatis 와 매핑

mybatis.config를 통해 alias를 사용했는데 죽어도 안됐는데 이 방법을 찾아서 다행이다.

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="Function">
	<sql id="checkEmpty">
		<bind name="isEmpty" value=":[@com.guivingAdmin.utils.CommonUtils@empty(#this)]"/>
		<bind name="isNotEmpty" value=":[@com.guivingAdmin.utils.CommonUtils@notEmpty(#this)]"/>
	</sql>
</mapper>
```



실제 사용

```
<sql id="searchWhere">
		<include refid="Function.checkEmpty"/>
		<if test="#fn=isNotEmpty, #fn(search.status)">
			...
		</if>
    (...)
	</sql>
  
  <select id="selectAllSettlement" resultMap="ResultMap.reservationResult">
		SELECT 
			...
		FROM
    ...
		WHERE ...
		<include refid="searchWhere"></include>
		
	</select>
```

#fn= 이부분마저 줄이고 싶었으나 일단 만족하는걸로 !

또한가지.. 이전에 resultMap 추가할 때 interface를 만들었는데 이번 작업을 하면서 불필요하단 것을 깨닫게 되었다.

namespace로 매핑이 되는 것이었구나..

jpa를 먼저 본다지만 mybatis구조도 좀 공부해보는걸로





마지막으로 javascript 부분도 좀 바꿨다.

위의 controller 단에 바뀌어 다음과 같은 플로우로 서버와 통신하게 된다.


1. 최초 페이지 접근 시 서버에서 검색 dto를 내려준다.

2. 클라이언트는 검색 dto를 form안에 params 라는 클래스 태그를 입력한다.

3. 검색을 진행할 시 각 params 클래스 인자들의 id를 토대로 class태그를 탐색한다. 

4. 탐색된 class의 값을 다시 params의 매치되는 요소에 부여한다.(여전히 params는 form안에 있다.)

5. 서버에 form 데이터를 전송하여 결과를 받은 뒤 다시 params의 값을 해당 요소들에 부여한다.

(검색 조건 UI 적용)


말은 길어졌지만 코드 보면 간단.


```

var elements={	
		init : function(){
			
			var _this = this;
			$('#btn_search').click(function(event) {
				_this.loadpage();
			});
			$('#btn_register').click(function(event) {
				event.preventDefault();
				location.href = $("#register_url").val();
			});
			$('.selectAll').click(function(event) {
				var flag = $('.selectAll').is(':checked');
				$('.select').each(function(){
					$('.select').prop('checked',flag);
				});
			});
			$('.countryIdx').click(function(){
				_this.setcity();
				$('.cityIdx').val("");
			});
			$('.cityIdx').click(function(){
				_this.setcompany();
				$('.companyIdx').val("");
			});
			
			$('.btn_update').click(function(){
				_this.update($(this).data('status'));
			});
			
			$('.page').click(function(){
				_this.gopage($(this).data('no'));
			});
			
			
			_this.getparams();
		},
		setcity : function (){
			var tempCountryIdx = $('.countryIdx').val();
			$('.cityIdx option').each(function(){				
				if ($(this).data("countryidx") == tempCountryIdx) {
					$(this).show();
				} 
				else{
					$(this).hide();					
				}
			});
		},
		
		setcompany : function(){
			var tempCityIdx = $('#cityIdx').val();
			
			$('.companyIdx option').each(function(){	
				if ($(this).data("cityidx") == tempCityIdx) {
					$(this).show();
				} 
				else{
					$(this).hide();					
				}
			});
		},
		getparams : function(){

			var _this = this;
			$(".params").each(function(){
				var elm = '.'+$(this).attr('id');
				var val = $(this).val()
				if(val){
					if($(elm).attr('type') == 'radio'){
						$('input:radio'+elm+':input[value=' + val + ']').attr("checked", true);
					}	
					else{
						$(elm).val(val);						
					}
				}
			});

			_this.setcity();	
			_this.setcompany();
			
		},
		setparams : function(){
			
			$(".params").each(function(){
				var elm = '.'+$(this).attr('id');
				var val;
				
				if($(elm).attr('type') == 'radio'){
					val = $("input:radio"+elm+":checked").val();
				}
				else{
					val = $(elm).val();
				}
				
				$(this).val(val);	
			});
			
		},
		loadpage : function(){
			var _this = this;
			_this.setparams();
			$("#searchForm").submit();
		},
		gopage : function(pageNo){
			$("#nowPage").val(pageNo);
			var _this = this;
			_this.loadpage();
		}
}
elements.init();
```

코딩 방식은 배민의 갓동욱님이 쓰신 책을 보고 따라함..

이 방식으로 기존 js도 좀 바꿔놨다.


내일부턴 기존 페이지들을 정산페이지와 같은 방식으로 페이징 및 검색 필터 적용이 필요하겠다.


인생 !!



