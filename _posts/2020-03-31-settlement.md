---
title: "본격 정산모듈작성2"
date: 2020-03-31 19:51:00 -0400
categories: mybatis dto ajax
---

어제와 이어서 정산 모듈 작성 중이다. 정확히는 정산 페이지 쯤 되려나..

mysql view를 사용하여 domain <-> dto 통신하기 까지가 어제의 개발 계획이었다.

아침에 view를 보니 잘못된 부분이 있어 좀 수정을 한 후에..

본격 페이지와 서비스 코드 작성

첫번째는 리스트를 작성하게 되었다.

역시 이동욱님의 책을 보며 최대한 따라하는 코드를 작성 아래와 같은 조회 코드가 나타났다.

서비스클래스 부분


```

@Service
@RequiredArgsConstructor
public class SettlementService {
	
	private final SettlementMapper settlementMapper;
	
	@Transactional(readOnly = true)
	public List<SettleListResponseDto> selectAllSettlement(SearchDto dto) throws Exception{
		return settlementMapper.selectAllSettlement(dto)
				.stream()
				.map(SettleListResponseDto::new)
				.collect(Collectors.toList());
	}
	public int selectAllCount(SearchDto dto) throws Exception{
		return settlementMapper.selectAllCount(dto);
	}
	public int saveSettlementStatus(List<SettleSaveRequestDto> dtoArray) throws Exception {
		return settlementMapper.updateSettlementStatus(
				dtoArray.stream()
				.map(SettleSaveRequestDto::toDomain)
				.collect(Collectors.toList()));
	}
}

```
selectAllSettlement 를 통해 domain 으로 얻은 결과를 dto로 변환한다.

(stream 사용법 숙지가 필요하겠다..)

아래 saveSettlementStatus 에서의 domain 변환 부분은 책에 따로 나와있지 않아 검색하여 작성하였다.

막상 작성하고보니 위와 크게 다르지 않았던것..

참고링크

https://dbbymoon.tistory.com/4


다음은 컨트롤러 부분


```

@RequiredArgsConstructor
@Controller
@RequestMapping({ "/settlement" })
public class SettlementController {
	protected Logger logger = LoggerFactory.getLogger(getClass());
	
	private final SettlementService settlementService;
	
	@RequestMapping({ "/list" })
	public ModelAndView listSettlement(@RequestParam(required=false) SearchDto dto) {
		ModelAndView mav = new ModelAndView("settlement/list.tiles");
		try {	

			//int total = settlementService.selectAllCount(dto);
			//PagingVO paging = new PagingVO();		
			//PagingVO vo = CommonUtils.getPageVO(paging, total);
			
			List<EnumValue> settleStatus = ValueUtils.getMapper().getAll().get("SettleStatus");
			
			List<SettleListResponseDto> resList = settlementService.selectAllSettlement(dto);
			
			logger.debug(resList.toString());
			//mav.addObject("paging", vo);			
			mav.addObject("search", dto);	
			mav.addObject("settleStatus", settleStatus);
			mav.addObject("result", resList);
			
		} catch (Exception e) {
			this.logger.error("ERROR - ", e);
		}
		return mav;
	}	
	
	@RequestMapping({ "/saveStatus" })
	@ResponseBody
	public int saveSettlementStatus(@RequestBody List<SettleSaveRequestDto> dtoArray) throws Exception {
		int result = 0;
		try {
			result = this.settlementService.saveSettlementStatus(dtoArray);
		} catch (Exception e) {
			this.logger.error("ERROR - ", e.getMessage());
			e.printStackTrace();
		}
		return result;
	}
}

```

listSettlement 부분은 딱히 볼 것이 없으나 추후에 검색 필터 및 페이징 부분 재작업이 필요하겠다..

아래 saveSettlementStatus 부분에서 조금 애먹었는데 이유는 List형태의 dto를 받아오고 싶었으나 계속된 매핑 실패

결국은 아래와 같은 ajax를 통해 매핑에 성공하였다.

```
update : function(status){	
			var _this = this;
			var dtoList=new Array();
			$(".select").each(function(){
				if($(this).is(':checked')){
					var temp= new Object()
					temp.resIdx = $(this).data('id');
					temp.status = status;
					dtoList.push(temp);
				}					
			});
			console.log(dtoList);
			
			$.ajax({
				url : "/settlement/saveStatus", //컨트롤러 URL
				data : JSON.stringify(dtoList),
				dataType : 'json',
				type : 'POST',
				contentType: 'application/json',
				success : function(data) {
					var result = data;
					if (result > 0) {
						alert('저장하였습니다.');
						_this.loadpage();
					}
					alert('저장 실패');
					_this.loadpage();
				},
				error : function(request, status, error) {
					alert("code:" + request.status + "\n" + "message:"
							+ request.responseText + "\n" + "error:" + error);
				}
			});			
		}
```

ajax 단 data : JSON.stringify(dtoArray), 부분이 핵심이었던 것

나는 애꿎은 Array 요소들에다가 json 파싱을 하고 있었다.


참고링크

https://hakurei.tistory.com/227



이렇게 세팅된 List dto들은 다시 위의 컨트롤러와 서비스를 타고 domain으로 컨버팅 되어 mybatis로 도달하게 된다.

```
<insert id="updateSettlementStatus" parameterType="java.util.List" useGeneratedKeys="true" keyProperty="resIdx">
		INSERT INTO TB_SETTLEMENT
		(...)
		
		VALUES
		<foreach collection="list" item="item" separator=" , ">
		(		
            #{item.one}, #{item.two}      
        )
        </foreach>
    </insert>
```
mybatis는 동적쿼리에 강하다.. 라고 들었다. 엄청 단편적인 예지만 어느정도 동의할 수 있는 부분.


오늘의 아쉬운점

1. 테스트 코드 사용 ... 얼른 시작해보자.

2. 공통 js 변환 . 책을 보고 따라해보았다. 내 개발 방식이랑 조금 달랐다. 의도했던 공통부분을 뽑아내는 것은 좀 변화를 줘야할듯



내일은 페이지 검색 필터 적용과 페이징 부분.

어떻게 한 번에 적용할 수 있을지 생각해보기.
