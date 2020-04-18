---
title: "url이미지 base64로 컨버팅"
date: 2020-04-18 19:00:00 -0400
categories: 
---


사용하진 못했지만
url로 로드된 이미지를 base64 스트림으로 바꾸는게 필요했다.
(이전에 파일 업로드 자료형을 base64로 바꿔버렸기 때문에)

s3 url에서 크로스오리진 이슈가 발생해서 이런 저런 세팅을 거쳤다.

s3버킷 권한 관리에서 CORS 란

```
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
<CORSRule>
    <AllowedOrigin>*</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedMethod>POST</AllowedMethod>
    <AllowedMethod>HEAD</AllowedMethod>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
    <AllowedHeader>*</AllowedHeader>
</CORSRule>
</CORSConfiguration>
```

일단 다 허용..!

그리고 cloudFront 세팅도 바꿔줘야하는데.. 이부분은 링크로 대체


https://stackoverflow.com/questions/12358173/correct-s3-cloudfront-cors-configuration


위의 세팅을 거치면 이미지 url이 로드된 뒤 base64로 컨버팅을 할 수 있게 된다.

다음은 자바스크립트 코드


```
$(window).on('load',function(){
	
	$("#frm img").each(function(){
		if(!$(this).attr('src'))
			return;
		
    	getDataUrl($(this));
    });
    
});

var getDataUrl = function (elm) {
  var canvas = document.createElement('canvas');
  var ctx = canvas.getContext('2d');
  var base64str;
  var img = new Image();
  img.crossOrigin = 'anonymous';
  img.src = $(elm).attr('src');
  img.onload = function(){
	  canvas.width = img.width;
	  canvas.height = img.height;
	  ctx.drawImage(img, 0, 0);
	  $(elm).attr('src',canvas.toDataURL('image/jpeg'));
  };
}


```

모든 요소들이 로드되는 이벤트에 콜백을 넣어두었다.
요약하자면 폼데이터 안에 있는 모든 이미지 URL을 base64로 바꾸란 의미

이미지를 컨버팅할 때에는 crossOrigin설정 가장 첫번째로 하고
이미지로드되는 시점에서 url를 컨버팅 할 수 있도록 했다.


비지니스 부분은 이전과 같다. 
변한거라곤 파일업로드 부분에서 empty 체크 정도..?

```

	@Transactional(rollbackFor = Exception.class)
	public void update(CompanyUpdateRequestDto request) throws Exception{
		Company company = request.toModel();
		companyMapper.updateCompany(company);
		if(!company.getFiles().isEmpty())
			companyMapper.saveCompanyDoc(company);		
	}
```

여기까지가 이번에 작성한 것 !!

끗










