---
title: "s3 파일 업로드 유틸"
date: 2020-03-24 18:00:28 -0400
categories: fileupload
---
maven dependency 추가

```
<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>aws-java-sdk-s3</artifactId>
  <version>1.11.452</version>
</dependency>

```

aws 보안 자격 증명 란 s3 액세스 ID , Key 발급 및 Region 체크
![11](https://user-images.githubusercontent.com/51354965/77408252-c73aea00-6dfa-11ea-977b-ec2e66a817ed.PNG)



S3 upload 클래스

```

import java.io.File;

import org.springframework.stereotype.Component;

import com.amazonaws.AmazonClientException;
import com.amazonaws.AmazonServiceException;
import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import com.amazonaws.services.s3.model.CannedAccessControlList;
import com.amazonaws.services.s3.model.ObjectMetadata;
import com.amazonaws.services.s3.model.PutObjectRequest;
import com.amazonaws.services.s3.transfer.TransferManager;
import com.amazonaws.services.s3.transfer.TransferManagerBuilder;
import com.amazonaws.services.s3.transfer.Upload;

@Component
public class S3Uploader {

	private AmazonS3 amazonS3;

	private String bucket="buketURL";

	private static String accessKey="accessKey";
	
	private static String secretKey="secretKey";
	
	private static String region="S3 Region";
	
	public S3Uploader() {
		BasicAWSCredentials awsCreds = new BasicAWSCredentials(accessKey, secretKey);
		this.amazonS3 =  AmazonS3ClientBuilder.standard()
	            .withCredentials(new AWSStaticCredentialsProvider(awsCreds))
	            .withRegion(region)
	            .build();
	}
  @Async
	public void uploadToS3(File file, String fname) {
		TransferManager tm = TransferManagerBuilder.standard().withS3Client(amazonS3).build();
		PutObjectRequest request;
		try {

			ObjectMetadata metadata = new ObjectMetadata();
			// metadata.setCacheControl("604800");// 60*60*24*7 일주일
			// metadata.setContentType("image/png");
			request = new PutObjectRequest(bucket, fname, file)
					.withCannedAcl(CannedAccessControlList.PublicRead);
			// amazonS3.putObject(request);
			Upload upload = tm.upload(request);

			upload.waitForCompletion();

		} catch (AmazonServiceException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (AmazonClientException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("bucketfname : "+ bucket+"/"+ fname);
		
	}
}

```

파일업로드 클래스

```

import java.io.File;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;

import javax.servlet.http.HttpServletRequest;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.multipart.MultipartHttpServletRequest;


@Component("fileUtils")
public class FileUtils {

	protected Logger logger = LoggerFactory.getLogger(this.getClass());

	@Autowired
	S3Uploader s3Uploader;
	
  	//하나의 field에 여러 파일이 업로드된 경우
	public void uploadMultiFilesHash(HttpServletRequest request, String uploadDir, HashMap paramMap)throws Exception {
		
		MultipartHttpServletRequest multipartHttpServletRequest = (MultipartHttpServletRequest) request;
		Iterator<String> iterator = multipartHttpServletRequest.getFileNames();

		List<MultipartFile> multipartFileList = null;
		String originalFileName = null;
		String originalFileExtension = null;
		String storedFileName = null;

		String realPath = request.getSession().getServletContext().getRealPath("/");
		String filePath = realPath + uploadDir;

		logger.debug("realPath[" + realPath + "]");
		logger.debug("filePath[" + filePath + "]");

		File file = new File(filePath);
		if (file.exists() == false) {
			file.mkdirs();
		}
		ArrayList<String> arrList = null;
		while (iterator.hasNext()) {
			multipartFileList = multipartHttpServletRequest.getFiles(iterator.next());
			arrList = new ArrayList<String>();
			String fileName =null;
			for(int i=0; i<multipartFileList.size(); i++) {
				if (!multipartFileList.get(i).isEmpty()) {
					MultipartFile multipartFile =  multipartFileList.get(i);
					originalFileName = multipartFile.getOriginalFilename();
					originalFileExtension = originalFileName.substring(originalFileName.lastIndexOf(".")); // 확장자
					storedFileName = CommonUtils.getRandomString() + originalFileExtension; // 랜덤파일명
	
					logger.debug("originalFileName[" + originalFileName + "]");
					logger.debug("storedFileName[" + storedFileName + "]");
	
					file = new File(filePath + storedFileName);
					try {
						multipartFile.transferTo(file);
					} catch (Exception e) {
						e.printStackTrace();
					} finally {
						fileName = multipartFile.getName();
						arrList.add(uploadDir + storedFileName);						
						s3Uploader.uploadToS3(file, uploadDir + storedFileName);
					}
				}
			}
			paramMap.put(fileName, arrList);			
		}
		
	}
	public void uploadFileHash(HttpServletRequest request, String uploadDir, HashMap paramMap)
			throws Exception {

		MultipartHttpServletRequest multipartHttpServletRequest = (MultipartHttpServletRequest) request;
		Iterator<String> iterator = multipartHttpServletRequest.getFileNames();

		MultipartFile multipartFile = null;
		String originalFileName = null;
		String originalFileExtension = null;
		String storedFileName = null;

		String realPath = request.getSession().getServletContext().getRealPath("/");
		String filePath = realPath + uploadDir;

		logger.debug("realPath[" + realPath + "]");
		logger.debug("filePath[" + filePath + "]");

		File file = new File(filePath);
		if (file.exists() == false) {
			file.mkdirs();
		}

		while (iterator.hasNext()) {
			multipartFile = multipartHttpServletRequest.getFile(iterator.next());
			if (!multipartFile.isEmpty()) {
				originalFileName = multipartFile.getOriginalFilename();
				originalFileExtension = originalFileName.substring(originalFileName.lastIndexOf(".")); // 확장자
				storedFileName = CommonUtils.getRandomString() + originalFileExtension; // 랜덤파일명

				logger.debug("originalFileName[" + originalFileName + "]");
				logger.debug("storedFileName[" + storedFileName + "]");

				file = new File(filePath + storedFileName);
				try {
					multipartFile.transferTo(file);
				} catch (Exception e) {
					e.printStackTrace();
				} finally {
					paramMap.put(multipartFile.getName(), uploadDir + storedFileName);
					s3Uploader.uploadToS3(file, uploadDir + storedFileName);
				}
			}
		}
	}
}

```


공통 유틸 클래스 ( 랜덤 스트링 생성)

```
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

import org.springframework.stereotype.Component;

import com.fasterxml.jackson.databind.ObjectMapper;

@Component
public class CommonUtils {
	public static String getRandomString() {
		return UUID.randomUUID().toString().replaceAll("-", "");
	}
	public static Map jsonToMap(String json) throws Exception{
		ObjectMapper mapper = new ObjectMapper();
		Map returnMap=null;
        try {

            // convert JSON string to Map
            Map<String, String> map = mapper.readValue(json, Map.class);

			// it works
            //Map<String, String> map = mapper.readValue(json, new TypeReference<Map<String, String>>() {});
            returnMap = map;

        } catch (IOException e) {
            e.printStackTrace();
        }
        return returnMap;
	}	
}


```
