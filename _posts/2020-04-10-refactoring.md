---
title: "multipartfile 힘들구나..2"
date: 2020-04-09 20:25:00 -0400
categories: multipartfile formdata
---
<br>
오늘은 좀 농땡이 쳤다.<br>
사실 뭐 다른이유도 있지만.<br>
<br>
다른 것 안하고 파일 업로드 부분만 줄기차게 봤는데<br>
업무용 컴퓨터가 아닌 개인컴퓨터로 작성하는 관계로 러프한 코드만 작성하는걸로<br>
<br>
원하던 구조는 다음과같다.<br>

```
class dto {
 field 1;
 field 2;
 ...
 List<fileVo> files;
}

class filevo{
  fileName;
  file ; 
  fileUrl;
}
```

각 파일은 자체의 이름과 파일 객체를 갖고 있고 이를 s3에 업로드 한 뒤 url을 반환 받아 도메인 모델로 변환한다.<br>
문제는 자바스크립트에서였다.<br>



```
form = new FormData($('#frm')[0]);
var fileArray = new Array();
$('.file').foreach(function(){
  var temp = new Object();
  temp.fileName = fileName;
  temp.fileName = $(this)[0].file[0];
  temp.fileUrl = '';
  fileArray.push(temp);
});
form.append(temp);
```



<br>
문제가 되는 부분은<br>
사용자 UI단에서 사용되는 필요한 문서 업로드 목록은 FileCode라는 Enum 집단에서 추출해온 것이며<br>
각 input file name필드 값은 enum의 key값과 매칭되어있다.<br>



```
foreach(data : FileCode)
<input type="file" class="files" name ="data.key"/>
```



이렇게 FileCode 분류된 코드 집단을 UI에서 자동적으로 뿌리고 각 파일이 어떤 용도인지 FileCode로 판별하는 것이 의도였다.<br>
<br>
이전에 사용했던 json 형태의 통신이 아닌 formData - multipart 통신을 사용하다보니<br>
리스트(배열)로 정의한 필드가 자바 객체와 매핑되지 않는 것이었다.<br>
오류는대략 'can not convert java string to java list' 요런거<br>
<br>
그래서 이래저래 찾아보니 form.append에 들어가는 부분은 다 string처리가 된다고 한다.<br>
(확실한건가..?)<br>
그렇다면 파일을 받기위해선 어떻게 해야할까<br>
<br>
1. 파일을 blob따위의 바이트 스트림으로 읽어들여 텍스트필드로 자바객체와 매핑한다.<br>
매핑된 자바객체는 outputstream으로 파일을 쓰고 s3로 업로드한다.<br>
<br><br>
2. MultiHttpServletRequest를 사용해 모든 Map<String,MultipartFile>형태의 객체를 불러온다.<br>
해당 map을 파일업로드 유틸로 업로드한 뒤 List<fileVO>형태로 받아온다.<br>
<br><br>
3. dto에 MultipartFile 필드를 html 폼의 input file name과 전부 동일하게 작성하여 1대1 매핑을 진행한다.<br>
dto의 필드를 List<Multipartfile>형태로 추출하여 업로드한 후 List<fileVO>형태로 반환받는다.<br>
<br><br>
아직 어떤 방법을 쓸지 결정하지 못했다. <br>
2,3번 방식은 dto와 file 객체가 따로 넘어와 뭔가 께름직하다.<br>
(해당 파일이 어떤 용도인지 확인할 방법이 getFileName()뿐이기 때문에)<br>
<br>
아니면 MultipartFile 자체에 어떤 정보를 담아서 보낼 수 있으면 더 쉬운 방법을 찾을수도 있겠다.<br>
오늘은 별다른 걸 못하고 이 부분에만 빠져있었네..<br>
다음주에 문규한테라도 물어보는걸로 ㅜㅠㅜ<br>
