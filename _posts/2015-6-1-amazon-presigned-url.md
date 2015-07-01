---
layout: post
title: Amazon S3 PUT using presigned URL with AWS Java SDK
---

In this blog post we will explore how Amazon SDK for Java can be used for generation of
presigned URL than can be used later for uploading files directly to S3 bucket.

##Topics Covered
* Generation of presigned URL with AWS SDK for Java
* Making uploded file public 
* Retrieving uploaded file back 

##Intro
The Amazon Simple Storage Service (S3) is one of the most widely used services of Amazon Web Services (AWS) stack. It is used for storing/retrieving files from user defined 'storage boxes' called buckets. 

When using S3 one of the ways of uploading file to S3 is the use of 'presigned URL': specially formatted URL string which is provided to the client. Using this URL client can upload file directly to S3 bypassing any mediator services.  


This is very convenient because:

* under the hood S3 internals can optimize the upload by using server that is the nearest to client
* software developer can remove any file-uploading routines from application code and concentrate on implementing business logic
* all the file handling logic is performed by S3 servers thus improving the overall performance of application


##Generation of Upload URL for PUTting Data to S3
In order to generate presigned URL two main classes come in to play:

* [AmazonS3Client](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/s3/AmazonS3Client.html) - main point of interaction between client code and S3 infrastructure
* [GeneratePresignedUrlRequest](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/s3/model/GeneratePresignedUrlRequest.html) - request object containig all information need for presigned URL generation  

```java
// create S3 client (for example from clientId/clientSecret)
AmazonS3Client s3Client = ...;
// URL generation request: method, bucket name and path in bucket
GeneratePresignedUrlRequest presignedUrlReq 
= new GeneratePresignedUrlRequest(bucket, "/dir", HttpMethod.PUT);
// setting additional params: expiration date , content-type, etc
long expiration = System.currentTimeMillis() + 1000_000L;
presignedUrlReq.withExpiration(new Date(expiration))
   				.withContentType("application/json");
// finally generating URL string
String urlStr = s3Client.generatePresignedUrl(presignedUrlReq);   				 
```
##Setting Permissions for Uploaded Files

By default all files uploaded to S3 are have 'private' visibility mode: only the owner of bucket with valid username and secret has access. It is reasonable default behaviour, but in some cases (for example for user uploaded content) it is desirable that uploaded files will be visible to the whole world right after the upload.

There are no direct method in GeneratePresignedUrlRequest for setting files public, but exploring the documentation one can find that all S3 requests support a group of custom headers whose names start with '**x-amz**'. The whole range of supported headers can be seen in dedicated [Headers](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/s3/Headers.html) class.

The header we are intrested is named '**x-amz-canned-acl**'. The whole range of values supported by this header listed in enum [CannedAccessControlList](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/s3/model/CannedAccessControlList.html). As one can see from comments in source code of above enum the value of 'Public-Read' is what we need: _If this policy is used on an object, it can be read from a browser without authentication_.

So the URl generation code can be improved with the lines:

```java
	if(isPublicResource) {
		// setting http request header:  
		// x-amx-canned-acl: 'public-read'
        generatePresignedUrlReq.addRequestParameter(
        	Headers.S3_CANNED_ACL, 
        	CannedAccessControlList.PublicRead.toString()
    	);
    }
```

##Running the code sample

There is a sample project at [GitHub](https://github.com/zstd/code.zstd.github.io/tree/master/aws-public-presigned-url) with functionality described above. 

Project can be build with Maven. It contains special 'integration-test' which uses generated presigned URLs to upload file with different permissions and then check their availability.

In order to run the sample:

* checkout/clone the project
* go to **src/conf** directory and copy *template.properties* to *local.properties* file 
* set your AWS credentials and bucket name in *local.properties*
* run PresignedUrlForPutGeneratorIT

