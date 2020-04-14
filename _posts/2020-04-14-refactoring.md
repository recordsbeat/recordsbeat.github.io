---
title: "rest api 응답과 테스트"
date: 2020-04-14 19:54:00 -0400
categories: restapi junit mockmvc
---
<br>
오늘 한 것<br>
<br>
1. RestController 리턴형태 ResposneEntity 사용하기<br>
<br>
2. FileUtils의 업로드 경로 자원화 및 객체 업로드<br>
<br>
3. 상세 조회 페이지작성<br>
<br>
+ 메소드 명 변경<br>
ex. select -> find / create -> save ..등<br>
<br>
<br>
<br>
ResposneEntity 의 응답과 ajax 반응형태는 대략 이렇게

```
@ResponseBody
	@PostMapping("/api/v1/company")
	public ResponseEntity<?> save(@RequestBody CompanySaveRequestDto request) throws Exception {
		ResponseEntity<?> result;
		try {
			companyService.save(request);
			result = new ResponseEntity<>(HttpStatus.OK) ;
		}
		catch(Exception e) {
			e.printStackTrace();
			result = new ResponseEntity<String>(e.getMessage(), HttpStatus.CONFLICT) ;
		}
		return result;
	}
```
<br>
ResponseEntity 인스턴스 생성당시 제네릭을 사용하여 전송<br>
<br>

```
$.ajax({
			url : "/api/v1/company",
		    data: JSON.stringify(jsonForm), 
			type : 'POST',
			contentType: 'application/json',
			success : function() {
				alert('저장하였습니다.');
			},
			error : function(request, status, error) {
				alert("something goes wrong");
				console.log("code:" + request.status + "\n" + "message:"
						+ request.responseText + "\n" + "error:" + error);
			}
		});
```
<br>
별다른 것 없이 http.OK 하면 200 떨어지면서 success 로 받아들임<br>
dataType : json 하면 응답형태 json 으로 받아서 사용못하더라능<br>
<br>
<br>
<br>
<br>
FileUtils 경로 자원화<br>

```

@Component("fileUtils")
public class FileUtils {

	protected Logger logger = LoggerFactory.getLogger(this.getClass());

	private static S3Uploader s3Uploader;
	
	private static String realPath;
	
	private static String companyUploadDir;
	
	@Autowired
	FileUtils(S3Uploader s3Uploader , ServletContext context){
		this.s3Uploader = s3Uploader;
		this.realPath = context.getRealPath("/");
	}
	
	public static String getRealPath() {
		return realPath;
	}
	
	@Value("${companyUploadDir}")
  private void setCompanyUploadDir(String companyUploadDir) {
		FileUtils.companyUploadDir = companyUploadDir;
  }
	public static String getCompanyFilePath() {
		return companyUploadDir;
	}
 }
```
<br>
properties 에 존재하는 경로를 static으로 가져오기 위해 setCompanyUploadDir 정의<br>
주의할 것은 클래스명.필드 를 명시해야 주입이된다. <br>
this.field = param 이거 안됨<br>
<br>
<br>
위를 사용해서 변환시에 파일 업로드 하는 걸로 바꿈<br>

```
public Company toModel() throws Exception{
		return Company.builder()
				.comName(comName)
				.comOwnerName(comOwnerName)
				.comBuildDate(comBuildDate)
				.comAddr(comAddr)
				.comAddrCity(comAddrCity)
				.comAddrState(comAddrState)
				.comBizNum(comBizNum)
				.files(FileUtils.uploadBase64Image(files,FileUtils.getCompanyFilePath()))
				.city(City.builder().city(city).build())
				.build();
	}
  
@Transactional(rollbackFor = Exception.class)
public void save(CompanySaveRequestDto request) throws Exception{
  Company company = request.toModel();
  companyMapper.saveCompany(company);
  companyMapper.saveCompanyCity(company);
  companyMapper.saveCompanyFee(company);
  companyMapper.saveCompanyDoc(company);		
}
```
<br>
files 필드 부분에서 직접적으로 업로드를 시도한다.<br>
io부분에 익셉션 처리가 되어있어서 throws를 추가.<br>
해서 아래 save메소드가 좀더 정갈하게 바뀜<br>
<br>
<br>
<br>
<br>
<br>
상세 조회 페이지 작성<br>
<br>
restcontroller로 작성하려다가 web인 것을 인지하고 다시 indexcontroller로 돌아감<br>

```
@GetMapping("company/{idx}")
	public ModelAndView findCompany(@PathVariable Long idx) {
		ModelAndView mav = new ModelAndView("company/detailView.tiles");
		try {
			mav.addObject("result",companyService.findByIdx(idx));
		} catch (Exception e) {
			this.logger.error("ERROR - ", e);
		}
		return mav;
	}
```
<br>
간단한 get형태의 uri . rest api 패턴과 같게한다.<br>
<br>
<br>
업로드한 파일목록을 가져오기위한 file ResultMap정의<br>

```
<resultMap type="com.guivingAdmin.domain.vo.DocFile" id="docFile">
		<id property="fileCode" column="FILE_CODE"/>		
		<result property="url" column="FILE_URL"/>		
	</resultMap>
```
<br>
DB 파일을 추가/수정할 때 신규 row를 insert하고 각 항목의 최상위건을 가져오도록 했었다.<br>
그랬더니 필요한 파일 목록을 UI에 그릴수 없어 쿼리문에서 필요 항목까지 한 번에 조회해오는 것으로 했다.<br>
<br>
이미 선언해놓은 FileCode를 UI단에 넘겨 그려놓고<br>
요청한 데이터의 filecode와 비교하며 그려볼까도 했지만 그러면 중첩 foreach가 생기게되는 것..<br>
docFile필드를 Map형태로 가져와서 Filecode를 key로 사용해 꺼내오는 방식을 사용해볼까<br>
<br>
아무튼 위 때문에 쿼리문이 상당히 지저분해짐<br>

```
<select id="findByIdx" resultMap="ResultMap.companyExt">
		SELECT CY.COM_IDX, CY.COM_NAME, CY.COM_OWNER_NAME, 
			CY.COM_BUILD_DATE, CY.COM_BIZ_NUM, CY.COM_ADDR, 
			CY.COM_BIZ_TYPE_CODE, CY.COM_BIZ_TYPE_NAME, 
			CY.COM_BIZ_LICENSE_URL, CY.COM_AUTH_CODE, CY.COM_ADDR_CITY, 
			CY.COM_ADDR_STATE, CY.COM_ADDR_POSTAL_CODE, 
			CY.COM_GARAGE_IMG_URL, CY.COM_CLEAN_IMG_URL, CY.COM_FIX_IMG_URL, CY.COM_ETC_IMG_URL,
			CD.FILE_CODE, CD.FILE_URL
		FROM 
		TB_COMPANY CY
			LEFT JOIN 
			(SELECT IFNULL(CD.COM_IDX,#{Idx}) AS COM_IDX,CF.FILE_CODE,CD.FILE_URL 
	      FROM 
	      (SELECT DOC_IDX, COM_IDX, FILE_CODE, FILE_URL, ISUSED 
	        FROM TB_COM_DOC CD,
	        (SELECT MAX(CD.DOC_IDX) AS MAX_IDX
	          FROM TB_COM_DOC CD
	          GROUP BY CD.DOC_IDX) SUB
	        WHERE CD.DOC_IDX = SUB.MAX_IDX)CD
	      RIGHT JOIN VW_COMPNAY_FILE CF ON CD.FILE_CODE = CF.FILE_CODE AND CD.COM_IDX = #{Idx})CD
			ON CD.COM_IDX = CY.COM_IDX
		WHERE CY.COM_IDX = #{Idx}
	</select>
```
<br>
Idx를 세군데나 쑤셔박았다. 내가 제일 싫어하는 스타일의 쿼리 ㅠㅠ<br>
UI에서 간편하게 매핑할 수 있는 걸 해봐야겠다.<br>
<br>
도메인 모델측에서<br>
filecode를 사용하여 docfile list를 만들고 매칭되는 데이터만 넘겨줘도 될 듯<br>
(애초에 이렇게 할 걸 쯧)<br>
<br>
<br>
마지막으로 처음 사용해본 컨트롤러 테스트<br>

```
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class IndexControllerTest {

	@Autowired
	private MockMvc mockMvc;

	@Test
	public void findCompanyTest() throws Exception {
		this.mockMvc.perform(get("/company/{idx}", 64))
			.andExpect(status().isOk())
			//.andExpect(jsonPath("$['result']", containsString("comName")))
			.andDo(print());
	}

}

```
<br>
아직 사용법을 잘몰라서 그런건지 view페이지에서 어떤 값이 매핑이 안되어서 나는 오류를 볼 수 가 없었다.<br>
이부분 알아내면 훨씬 편하지 싶은데..<br>
<br>
<br>
내일은 수정 기능까지 <br>
<br>
되도록 일관된 dto를 사용하는것이 제일 좋으니 그쪽으로 유도해보는걸로<br>





