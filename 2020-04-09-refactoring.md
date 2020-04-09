---
title: "multipartfile 힘들구나.."
date: 2020-04-09 20:25:00 -0400
categories: multipartfile formdata
---

<br>
오늘 진행한 세가지<br>
<br>
1. lazyloading 을 전략적으로 사용해보기 (모든 쿼리에 outer join 걸기에는 시간이 너무 많이 걸린다.)<br>
<br>
2. Enum 클래스 부모 코드와 부모코드에 따른 공통 그룹 반환하기<br>
<br>
3. Multipartfile DTO에 필드와 함께 전송하기 (FormData , Ajax)<br>
<br>
<br>
시작하기 전에..<br>
<br>
도메인 모델링이란 것을 계속 보다가 알게 된 것<br>
도메인 모델 - JPA에서 entity랑 비슷한 개념이다.<br>
루트 엔티티 - 애그리거트 루트 . 최상위가되는 데이터 집단. <br>
엔티티별로 dao를 분리할 필요가 없다. 애그리거트 기준으로 dao를 컨트롤 하면 될 것<br>
이래저래 읽고 이해하려고 한 부분<br>
<br>
<br>
첫번째<br>
<br>
<br>
이전에 1차적으로 로드되는 연관 객체는 outer join을 2차적으로 로드되는 객체는 lazyload를 사용한다고 했었다.<br>
그러나 막상 써보니 모두 즉시 로드 되는 것이 아닌가 ?<br>
당황을 금치 못해 1차 로드 되는 연관 객체의 resultMap을 따로 선언하고 이를 extends . <br>
완벽히 로드되어야할 때 (연관객체로 로드 된 것이 아닌 주체가 될 때)만 lazyload를 하도록 했다.<br>
<br>

```
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
    	<collection property="citys"  fetchType="lazy" column="COM_IDX" javaType="java.util.List" select="com.guivingAdmin.dao.CityMapper.selectCityByComIdx"/>		 
	</resultMap>
```
<br>
<br>
위 company는 reservation 따위에 1차적으로 로드될 때 쓰이는 resultMap<br>
밑 companyExt는 company 모델 자체를 불러올 때 lazyload를 하도록 만듦<br>
<br>
근데 알고보니<br>
mybatis에서 aggressiveLazyLoading이라는 세팅 값이 있다. <br>
<br>
'활성화되면 모든 메서드 호출은 객체의 모든 lazy properties 을 로드한다. 그렇지 않으면 각 property 가 필요에 따라 로드된다. (lazyLoadTriggerMethods 참조).'<br>
<br>
라고 나와있다.<br>
<br>
하여 다음과 같이 mybatis-config 작성<br>

```
<settings>
         <setting name="lazyLoadingEnabled" value="true" />
         <setting name="aggressiveLazyLoading" value="false" />
         <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString" />
		<setting name="mapUnderscoreToCamelCase" value="true"/>
	</settings>
```
<br>
lazyLoadingEnabled을 true로 세팅 (fetchType으로 대체 될 수 있다고 한다.)<br>
aggressiveLazyLoading false 세팅 -> 직접적으로 호출 될 때만 쿼리를 실행하도록<br>
lazyLoadTriggerMethods 지연로딩을 야기하는 메소드 명시라고 한다. <br>
<br>
위의 세팅을 하고 테스트 코드를 돌려봤다.<br>

```

	@Test
	public void testSelect() throws Exception {
		SettlementListSearchDto search = SettlementListSearchDto.builder()
				.build();
		PagingDto paging = PagingDto.builder().nowPage(1)
				.cntPerPage(10)
				.total(10)
				.build();
		List<Settlement> sList = sm.selectSettlementList(search,paging);
		System.out.println("sListsListsList : " + sList.get(0).getReservation().getVehicle());
	}
```
<br>
왠걸 .. 안된다..<br>
이걸 갖고 몇 번을 헤맸다. 혹시나 테스트코드에서만 정상작동 안하는 것인가 해서 서버를 직접돌리고도 해봤다.<br>
아무튼 결과적으로는 의도한대로 lazyload가 진행된다. 왜인지 모르겠는데 toString()이 불려지만 lazyload가 아니라<br>
전체 연관객체 로드가 진행된다. lazyLoadTriggerMethods에 toString넣으면 되는줄 알았는데 아닌가보다..<br>
<br>
일단 이부분은 이렇게 넘어가는걸로<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
두번째<br>
<br>
업체목록 조회는 저번에 dto를 통해서 작성했고 이번에는<br>
업체등록 수정이 필요하다. 업체등록 수정은 dto와 domain model간 통신을 목표로하게 되었고<br>
그 중 에그리거트라 할 수 있는 company model을 작성, 그에 필요한 vo(Value Object)로 file을 작성하게 되었다.<br>
<br>
<br>

```

@NoArgsConstructor
@Getter
@ToString
public class DocFile {

	FileCode fileCode;
	File file;
	String url;
	
	
	@AllArgsConstructor
	@ToString
	public enum FileCode implements CodeEnumExt{

		DTI("500C01",FileType.COMPANY,"DTI, Department of Trade in Industry"),
		MPT("500C02",FileType.COMPANY,"Mayors Permit"),
		COR("500C03",FileType.COMPANY,"COR, Certificate of Registration from the Bureau of Internal Revenue"),
		DOT("500C04",FileType.COMPANY,"Department of Tourism Accreditation of the company"),
		LTF("500C05",FileType.COMPANY,"LTFRB franchise permit");

		private final String code;
		private final FileType parentCode;
		private final String comment;

		@Override
		public String getComment() {
			return comment;
		}

		@Override
		public String getKey() {
			return name();
		}

		@Override
		public String getCode() {
			return code;
		}
		@Override
		public FileType getParentCode() {
			 return parentCode;
		}
		
		public static List<FileCode> getEnumByGroup(FileType parentCode){
			return EnumGroup.getEnumByGroup(FileCode.class, parentCode);
		}
		
		public static List<FileType> getParentEnum(){
			return EnumGroup.getParentEnum(FileType.class);
		}
		
		public static class TypeHandler extends CodeEnumTypeHandler<FileCode> {
			public TypeHandler() {
				super(FileCode.class);
			}
		}
		
		
		
		
		@ToString
		@AllArgsConstructor
		public enum FileType implements CodeEnum {

			COMPANY("500C", "업체자료"),
			DRIVER("500D", "드라이버자료"),
			CUSTOMER("500U", "고객자료"),
			VEHICLE("500V", "차량자료");

			private final String code;
			private final String comment;

			@Override
			public String getComment() {
				return comment;
			}

			@Override
			public String getKey() {
				return name();
			}

			@Override
			public String getCode() {
				return code;
			}

			public static class TypeHandler extends CodeEnumTypeHandler<FileType> {
				public TypeHandler() {
					super(FileType.class);
				}
			}

		}
		
	}
}

```
<br>
최 하단 FileType은 공통코드상 File의 귀속 주체를 뜻하고 각 FileType당 갖고 있을 FileCode를 Enum으로 다시 명시하였다.<br>
그리고 생각해보는데 UI상 필요한 서류를 명시에 주기 위해서는<br>
FileCode에서 업체, 드라이버, 고객 등 귀속 주체에 따른 FileCode 그룹이 필요했고,<br>
parentCode의 EnumClass 또한 가져올 수 있어야 했다.<br>
<br>
<br>
그래서 부모코드를 갖는 Enum 클래스를 따로 명시하도록 하고 자기자신의 클래스와 부모코드를 넣으면 해당 Enum Group을 뽑을 수 있도록 했다.<br>

```

public interface CodeEnumExt extends CodeEnum{
	public <T extends Enum<T> & CodeEnum> T getParentCode();
}

```
<br>
code, parentCode, comment를 갖는 Enum class를 인터페이스로 규정<br>
<br>
<br>
<br>

```

import java.util.EnumSet;
import java.util.List;
import java.util.stream.Collectors;

import com.guivingAdmin.interfaces.CodeEnum;
import com.guivingAdmin.interfaces.CodeEnumExt;

public class EnumGroup {
	public static <T extends Enum<T> & CodeEnumExt, E extends Enum<E> & CodeEnum> List<T> getEnumByGroup(Class<T> enumClass, E parentCode){
		return EnumSet.allOf(enumClass)
				.stream()
				.filter(type -> type.getParentCode().equals(parentCode))
				.collect(Collectors.toList());
	};
	
	public static <E extends Enum<E> & CodeEnum> List<E> getParentEnum(Class<E> enumClass){
		return EnumSet.allOf(enumClass)
		.stream()
		.collect(Collectors.toList());
	};
	
}
```
<br>
전역으로 사용될 수 있는 static 메소드를 넣어 위에서 FileCode가 그룹 Enum 리스트와 부모 enum 리스트를 반환 할 수 있도록 했다.<br>
처음으로 제대로 제네릭(와일드카드) 자료형과 stream을 써본듯 하다.(뿌듯한 부분)<br>
<br>
위의 코드를 테스트 해보면 다음과 같다.<br>
<br>


```
public class EnumGroupTest {
	
	@Test
	@Ignore
	public void testEnumGroup() {
		List<FileCode> list = FileCode.getEnumByGroup(FileType.COMPANY);
		list.stream()
		.forEach(x -> System.out.println("element : " + x));
	}
	
	@Test
	@Ignore
	public void testParentEnum() {
		FileCode.getParentEnum().stream()
		.forEach(x -> System.out.println("element : " + x));
	}
	
	@Test
	public void testEnumGroupCity() {
		List<CityEnum> list = CityEnum.getEnumByGroup(CountryEnum.KOREA);
		list.stream()
		.forEach(x -> System.out.println("element : " + x));
	}
	
	@Test
	public void testParentEnumCity() {
		CityEnum.getParentEnum().stream()
		.forEach(x -> System.out.println("element : " + x));
	}

}
```
<br>
FileCode를 작성하다보니 CityEnum도 같은 조건이라 동일하게 적용했다.<br>
테스트 잘됨(뿌듯2)<br>
<br>
<br>
<br>
<br>
<br>
마지막<br>
<br>
MultipartFile 업로드하기.<br>
조건은 다음과 같다.<br>
새로운 FormData를 생성,<br>
텍스트 필드와 MultifilePart 필드를 ajax로 서버에 전송한다.<br>
서버는 DTO형태의 자료형으로 텍스트필드와 Multipartfile을 받는다.<br>
<br>
하지만 실패...<br>
<br>
이래저래 움직여보다가 알게 된 것<br>
<br>
-RequestBody 가 아닌 Dto 클래스는 Setter가 없으면 매핑이 되지 않는다.<br>
이거 몰라서 진짜 한참 뻘짓했다. MultipartFile이 안넘어오는 건 이해했는데<br>
요놈 못잡아서 시간날림..<br>
<br>
아무튼 dto는 이렇다.<br>

```

@Getter
@Setter
@NoArgsConstructor
@ToString
public class CompanySaveRequestDto {
	private String comName;
	private String comOwnerName;
	private String comBuildDate;
	private String comAddr;
	private String comAddrCity;
	private String comAddrState;
	private CityEnum city;
	private String comBizNum;
	//private String comBizLicenseUrl;
	private String comBizTypeCode;
	private String comBizTypeName;
	private String comAddrPostalCode;
	/*
	private String comGarageImgUrl;
	private String comCleanImgUrl;
	private String comFixImgUrl;
	private String comEtcImgUrl;
	*/
	//private MultipartFile[] files; 
}

```
<br>
실패로 인해 
```
private MultipartFile[] files; 
```
부분이 주석처리 되어있다.ㅜㅜ<br>
<br>
<br>
무튼 원래 하고 싶던 건<br>
위에 써놓은 DocFile을 List형태로 받아 업로드하고 DB에 넣는 건데 (물론 모델로 변환해서..)<br>
File 자료형으로 바로 매핑은 되지 않았다.<br>
<br>
그래서 선회할까하는 방식은<br>
<br>
MultipartFile을 받아서 차례로 S3에 업로드한다.<br>
각 파일들의 이름은 FileCode로 명시되어있고(MTI, COR같은..)<br>
이를 DocFile 필드에 url과 같이 채워넣어서 리스트 형태로 반환한다.<br>

```
files.upload()//List<FileDoc>형태
.stream()
.map(FileDomain::toDomain)
.collect(Collectors.toList);
```
요정도 되려나..<br>
<br>
<br>
<br>
오늘 실패해버린 파일업로드 별 거 아닌것 같은데 엄청 속썩인다.<br>
내일 다시 도전해보는걸로..

