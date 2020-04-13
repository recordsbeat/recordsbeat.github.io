---
title: "파일 업로드와 다중 insert"
date: 2020-03-24 13:28:28 -0400
categories: base64 mybatis insert
---
<br>
저번주에 골머리를 썩혔던 파일 업로드 부분과<br>
데이터 insert 부분을 진행했다.<br>
<br>
각설하고 시작하자면 파일업로드 부분에서 multipartFile은 일단 버려두었다.<br>
기존에 js단에서 업로드된 이미지를 base64 포맷으로 컨버팅하여 UI단에 보여주는 것을 이용하여<br>
base64 스트림을 사용한 이미지 업로드를 구현하였다.<br>

```

@Data
public class Base64File{
	private FileCode fileCode;
	private String base64Str;
	
	@Builder
	public Base64File(FileCode fileCode, String base64Str) {
		super();
		this.fileCode = fileCode;
		this.base64Str = base64Str;
	}
}
```


일단 편의를 위해 lombok data 어노테이션 사용..<br>
<br>
<br>
<br>
DTO 


```

@Data
public class CompanySaveRequestDto {

	private String comName;
	private String comOwnerName;
	private String comBuildDate;
	private String comAddr;
	private String comAddrCity;
	private String comAddrState;
	private CityEnum city;
	private String comBizNum;
	private String comBizTypeCode;
	private String comBizTypeName;
	private String comAddrPostalCode;
	
	private List<Base64File> files;
	
	
	@Builder
	public CompanySaveRequestDto(String comName, String comOwnerName, String comBuildDate, String comAddr,
			String comAddrCity, String comAddrState, CityEnum city, String comBizNum, String comBizTypeCode,
			String comBizTypeName, String comAddrPostalCode, List<Base64File> files) {
		super();
		this.comName = comName;
		this.comOwnerName = comOwnerName;
		this.comBuildDate = comBuildDate;
		this.comAddr = comAddr;
		this.comAddrCity = comAddrCity;
		this.comAddrState = comAddrState;
		this.city = city;
		this.comBizNum = comBizNum;
		this.comBizTypeCode = comBizTypeCode;
		this.comBizTypeName = comBizTypeName;
		this.comAddrPostalCode = comAddrPostalCode;
		this.files = files;
	}
	
	public Company toModel(List<DocFile> fileList) {
		List<City> cityList = new ArrayList<City>();
		cityList.add(City.builder().city(city).build());
		return Company.builder()
				.comName(comName)
				.comOwnerName(comOwnerName)
				.comBuildDate(comBuildDate)
				.comAddr(comAddr)
				.comAddrCity(comAddrCity)
				.comAddrState(comAddrState)
				.comBizNum(comBizNum)
				.files(fileList)
				.citys(cityList)
				.build();
	}
}
```


리스트 형태로 Base64File 필드를 갖는다.<br>
<br>
<br>
ajax 부분



```
put : function(){

		var tempform = JSON.stringify($('#frm').serializeObject());
		var jsonForm = JSON.parse(tempform);
		
		var fileList = new Array();
		$(".files").each(function(){	
			if(!$(this).next('img').attr('src'))
				return;
			
			var temp=  new Object();
			temp.fileCode = $(this).attr('id');
			temp.base64Str = $(this).next('img').attr('src');
			fileList.push(temp);
		});
		
		jsonForm.files = fileList;
		
		console.log(jsonForm);
		

		$.ajax({
			url : "/api/v1/company",
		    data: JSON.stringify(jsonForm), 
		    dataType : 'json',
			type : 'PUT',
			contentType: 'application/json',
			success : function(data) {
				var result = data;
				if (result) {
					alert('저장하였습니다.');
				} else {
					alert('저장 실패');
				}
			},
			error : function(request, status, error) {
				console.log("code:" + request.status + "\n" + "message:"
						+ request.responseText + "\n" + "error:" + error);
			}
		});
		
	}
```

<br>
조금 지저분해서 안타깝지만 대략 이러하다.<br>
이미지를 업로드할 때 자동으로 img 태그에 src로 주어진 base64를 파일코드와 함께 오브젝트화 한다.<br>
UI의 텍스트 필드를 json 오브젝트로 파싱한 후 fileList 배열을 추가<br>
다시 json string으로 전송한다.<br>
<br>
<br>
<br>
파일 유틸부분<br>


```
@Autowired
FileUtils(S3Uploader s3Uploader , ServletContext context){
  this.s3Uploader = s3Uploader;
  this.context = context;
}
public static List<DocFile> uploadBase64Image(List<Base64File> base64File,String target) throws Exception{
  String prefixPath = context.getRealPath("/");
  String filePath =  prefixPath+"/"+target;
  List<DocFile> fileList = new ArrayList<DocFile>();
  /*
  List<DocFile> list = base64File.stream()
    .map(x-> Base64ToImgDecoder.decoder(x.getBase64Str(), filePath))
    .peek(x-> s3Uploader.uploadToS3(x, target + x.getName()))
    .map(x-> DocFile.builder().fileCode())
    .collect(Collectors.toList());
  */

  for(Base64File item : base64File) {

    File file = Base64ToImgDecoder.decoder(item.getBase64Str(), filePath);
    s3Uploader.uploadToS3(file, target + file.getName());
    DocFile temp = DocFile.builder()
        .fileCode(item.getFileCode())
        .url(target + file.getName())
        .build();
    fileList.add(temp);
  }
  return fileList;

}

public class Base64ToImgDecoder {
	public static File decoder(String base64, String target) {
	        String[] strings = base64.split(",");
	        String extension;
	        String rdStr = CommonUtils.getRandomString();
	        
	        switch (strings[0]) {//check image's extension
	            case "data:image/jpeg;base64":
	                extension = ".jpeg";
	                break;
	            case "data:image/png;base64":
	                extension = ".png";
	                break;
	            default://should write cases for more images types
	                extension = ".jpg";
	                break;
	        }
	        //convert base64 string to binary data
	        byte[] data = DatatypeConverter.parseBase64Binary(strings[1]);

	        String path = target  + rdStr + extension;
	        File file = new File(path);
	        try (OutputStream outputStream = new BufferedOutputStream(new FileOutputStream(file))) {
	            outputStream.write(data);
	        } catch (IOException e) {
	            e.printStackTrace();
	        }
	        
	        return file;
	}
}


```
<br>
<br>
FileUtils<br>
context path를 가져오는 부분을 더 간단히 하고 싶었으나 실패.. fileutils 클래스에 의존성 주입하였다.<br>
<br>
uploadBase64Image<br>
스트림을 써서 리턴하고 싶었으나 File 형태로 컨버팅되기전 fileCode를 어떻게 지니고 있을지 몰라 일단 진행..<br>
<br>
Base64ToImgDecoder<br>
인터넷에서 긁어온 base64toFile 부분 알아서 확장자도 확정해줘서 좋다.<br>
<br>
<br>
서비스 부분<br>

```
@Transactional(rollbackFor = Exception.class)
	public boolean putCompany(CompanySaveRequestDto request) throws Exception{
		List<DocFile> fileList = FileUtils.uploadBase64Image(request.getFiles(), companyUploadDir);
		Company company = request.toModel(fileList);
		companyMapper.putCompany(company);
		companyMapper.putCompanyCity(company);
		companyMapper.putCompanyFee(company);
		companyMapper.putCompanyDoc(company);
		
		return true;
	}
```

<br>
트랜잭셔널 어노테이션 추가..<br>
toModel 부분에 fileList 파라미터가 들어간게 아쉽다. 파일업로드를 toModel 부분에서 진행하는게 좋을까싶다.<br>
(realPath를 구하는데 문제가 있지만 생각해보니 Context를 사용해 bean에 realPath를 구해놓고 계속 가져다 쓰면 되지 싶다.)<br>
아니면 company 모델에서 역으로 Base64File리스트를 받아 업로드한 후 내부의 DocFile list로 변환하는게 맞을거 같기도하다.<br>
<br>
쿼리부분


```
<insert id="putCompany" useGeneratedKeys="true"	keyColumn="COM_IDX" keyProperty="comIdx" parameterType="com.guivingAdmin.domain.Company">
		INSERT INTO TB_COMPANY
		(
			COM_NAME,
			COM_OWNER_NAME,
			COM_BUILD_DATE,
			COM_BIZ_NUM,
			COM_ADDR,
			COM_AUTH_CODE,
			COM_ADDR_CITY,
			COM_ADDR_STATE
		)
		VALUES(
			#{comName},
			#{comOwnerName},
			#{comBuildDate},
			#{comBizNum},
			#{comAddr},
			SUBSTRING(MD5(RAND()) FROM 1 FOR 8),
			#{comAddrCity},
			#{comAddrState}
		);
	</insert>
```


키 칼럼을 통해 insert 된 id들 가져온다. <br>
company 모델에 setter가 없었는데 작동한걸 보니 내부적으로 다르게 동작하는가 싶다.<br>
(insert후 다시 select한다던가..)<br>
중간에 랜덤 스트링을 반환하는 부분은 날짜나 시퀀스등과 혼합사용해야 혹시모를 중복을 피할수 있겠다.<br>
(이부분이 company 모델에서 내부 비지니스 처리 로직으로 들어가면 좋겠다.)<br>
<br>
<br>
<br>
테스트 코드

```
@Test
	public void putCompanyTest()  throws Exception {
		List<Base64File> files = new ArrayList<Base64File>();
		files.add(Base64File.builder()
				.fileCode(FileCode.MPT)
				.base64Str("data:image/jpeg;base64,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")
				.build());
		CompanySaveRequestDto request = CompanySaveRequestDto.builder()
				.files(files)
				.build();
		cs.putCompany(request);
	}
```

<br>
MockMvc와 restTemplate를 사용하여 컨트롤러 단부터 서비스까지의 테스트 코드 숙지가 필요하겠다.<br>
service까지는 대략적으로 테스트 코드를 작성하는데<br>
controller부터 (특히 파일업로드 쪽) 에서 조금 복잡해보여 제대로 시도해보지 않았다.<br>
<br>
<br>
좀 지치게하는 일이지만<br>
그로인해 이제까지 고민했던부분의 문제점과 개선방향에 대해서 다시 생각해볼 필요가 있다.<br>
뛰지말고 걷더라도 멈추지 않는걸로



