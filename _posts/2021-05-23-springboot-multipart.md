---
layout:     post
title:      Vue.js의 axios를 이용한 SpringBoot로 이미지 파일 업로드하는 법
author:     Soo-young Hwang
tags: 		Spring-Boot Vue.js 
subtitle:  	
category:   study
---


### `Vue.js` `axios`를 이용한 `SpringBoot`로 이미지 파일 업로드하는 법
- 코드만 기록했고 코드에 대한 간략한 설명만 기술함. 
- 버전 : `vue.js` 2버전, `Spring boot` 2.3.4 버전

</br>
### 코드

#### 백엔드 (`Spring-boot`)
<blockquote>컨트롤러</blockquote>
```java
@PostMapping
  ResponseEntity<?> create(SendNumberDto dto) {
    CommonResponse<?> response = sendNumberService.registerSendNumber(dto);
    return new ResponseEntity<>(response, HttpStatus.OK);
  }
```
- `@RequestBody`를 붙이면 안 된다. 
    - `@RequestBody`는 클라이언트가 전송하는 <strong>`Json(application/json)` 형태</strong>의 `HTTP Body` 내용을 `Java Object`로 변환시켜주는 역할을 하기 때문에, `multipartFile` type이 있을 때 변환하는데 다른 `serializer`가 필요하다는 에러를 던진다. 

<blockquote>DTO</blockquote>
```java
@Data
@Builder
@AllArgsConstructor
public class SendNumberDto {
  private Long id;
  private Long memberId;
  private String name;
  private String sendNumber;
  private String topYn;
  private String desc;
  private String status;
  private String rejectReason;
  private Long authNumber;
  private Long fileId;
  private String originFileName;
  private String fileName;
  private String delYn;
  private MultipartFile files;
  private String createdBy;
  private LocalDateTime createdDate;
  private String lastModifiedBy;
  private LocalDateTime lastModifiedDate;
}
```
-  `null`이 허용되지 않는 `primitive` type `long` 에서 `Long`으로 수정했음. 
- `java.lang.IllegalStateException: No primary or default constructor found for class` 라는 에러가 났었다. 기본생성자가 없다는 에러였다. `@Data @Builder` 만 있었을 때 생겼던 에러였다.   
    이를 해결하기 위해, `@NoArgsConstructor`를 이용하여 기본 생성자를 만들어줬더니 빌더에 꼭 필요한 생성자가 있는데 위의 어노테이션으로 요소가 하나도 없는 생성자로 대체되어서 에러메시지가 또 생겼다. 그래서 `@AllArgsConstructor`를 붙여 모든 요소가 포함되는 생성자를 생성하도록 명시했다.

<br/>

#### 프론트엔드 (`Vue.js`)
<blockquote>axios를 이용해 post request하는 부분</blockquote>
```javascript
import { serialize } from 'object-to-formdata'

let formData = serialize(this.sendNumber)

if(isValid && this.sendNumber.memberId){
  if(confirm('등록하시겠습니까?')){
    this.$axios.post('/api/send-number',formData , {
      headers:{
          'Authorization' : `Bearer ${JSON.parse(localStorage.getItem('user')).accessToken}`,
          'Content-Type': "multipart/form-data",
          contentType: false, 
          processData: false,
        }
      })
    .then(() => {
//... 생략
```
- [object를 `formdata`로 변환](https://stackoverflow.com/questions/22783108/convert-js-object-to-form-data)하고 form형태로 request 요청함.
- vue.js 에서 object 그대로 (this.sendNumber)을 쓰면 에러났었음
    - `the request was rejected because no multipart boundary was found`
    - 이 에러를 해결하기 위해 [`import { serialize } from 'object-to-formdata'`](https://www.npmjs.com/package/object-to-formdata)를 임포트해주고 `serialize` 함. (`npm install object-to-formdata`)