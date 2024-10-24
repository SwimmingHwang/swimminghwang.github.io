---
layout:     post
title:      Spring boot API Server Post Request 시 Response data 한글깨짐
category:   Spring Boot
tags: 		HTTP-Request
---


### 환경
- Spring boot 2.1.9.REALEASE
- IntelliJ
- Java 1.8

## 문제
Spring boot API Server Post Request 시 Response data 한글 깨짐 현상

- 프로세스
  - client 에서 -> /api/at-msg  post request로 -> 컨트롤러에서 request 처리 -> 처리 중에 내부적으로 /api/v1/at-msgs post request

### 시도 1. `responseBodyConverter`와 `characterEncodingFilter` 빈객체 등록 및 charset 변경

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.PropertySource;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.converter.StringHttpMessageConverter;
import org.springframework.web.filter.CharacterEncodingFilter;
import lombok.extern.slf4j.Slf4j;

import javax.servlet.Filter;
import java.nio.charset.Charset;

@Slf4j
@PropertySource("classpath:kafka.properties")
@SpringBootApplication
public class APIApplication {
  public static void main(String[] args) {
    SpringApplication.run(APIApplication.class, args);
  }

  @Bean
  public HttpMessageConverter<String> responseBodyConverter() {
    return new StringHttpMessageConverter(Charset.forName("UTF-8"));
  }

  @Bean
  public Filter characterEncodingFilter() {
    CharacterEncodingFilter characterEncodingFilter = new CharacterEncodingFilter();
    characterEncodingFilter.setEncoding("UTF-8");
    characterEncodingFilter.setForceEncoding(true);
    return characterEncodingFilter;
  }
}
```
application 클래스에 빈 등록 코드를 작성하고 setting을 한다.

- `responseBodyConverter`는 결과를 출력시에 강제로 UTF-8로 설정하는 역할을 하며 `characterEncodingFilter`는 POST 요청시에 한글이 깨지는 문제를 보완함.


**하지만 중복으로 빈이 생성되었다는 에러가 난다.**

```text
2020-11-21 17:48:06.356 ERROR 15208 --- [           main] o.s.b.d.LoggingFailureAnalysisReporter   :

***************************
APPLICATION FAILED TO START
***************************

Description:

The bean 'characterEncodingFilter', defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/HttpEncodingAutoConfiguration.class], could not be registered. A bean with that name has already been defined in com.xxxxx.api.APIApplication and overriding is disabled.

Action:

Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true


> Task :intern-kafka-api:bootRun FAILED
```
너무 오래된 게시글을 봤었다. *다음부터 부트 버전에 맞게 검색 결과 참고하자.
에러 로그를 참고하여 2.1.9 버전에는 `HttpEncodingAutoConfiguration` 클래스가 auto configuration에 속해있어
`application.properties` 에서 설정을 세팅해 줄 수 있다고 한다.

그래서 두번째 시도를 했다.

### 시도2. `application.properties` 에 속성 추가

# Encoding UTF-8
spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
spring.http.encoding.force=true
했는데 동일하게 한글깨짐 현상이 남아있었다.

대체 어디서 한글이 깨지는지 범위를 줄여보고자 했다.
그래서 일단 return 한글로 테스트를 해봤다.
```java
    @RequestMapping(value="/"/*, produces="text/plain;charset=UTF-8"*/)
    @ResponseBody
    public String root() {
        return "한글";
    }
```
한글이 잘 response 되었고, 로그에도 한글이 잘 나오는 것을 확인했다.

```text
: POST "/", parameters={}
: Mapped to public java.lang.String com.xxxxx.api.controller.AtController.root()
: Using 'text/plain', given [*/*] and supported [text/plain, */*, text/plain, */*, application/json, application/*+json, application/json, application/*+json]
: Writing ["한글"]
: Completed 200 OK
```
---

두 가지 시도를 하면서, 문제 범위를 정확히 파악을 못 해 해결을 하지 못하고 있다는 생각이 들었다.
그래서 문제 범위를 좁혀나가기 시작했다.


우선 POST 요청에 문제가 있는지 먼저 확인을 했다.
OpenAPI 테스트와 HttpClient를 활용한 코드 두개로 테스트를 했다.

*웹에서 post요청시 문제가 있는지 확인하기 위해.
open API에서 한글이 잘 return된다면 HttpClient 요청시 문제가 있음을 알 수 있을 거라고 생각했다.


```text
: POST "/api/at-msg", parameters={}
: Mapped to public java.lang.String com.xxxx.api.controller.AtController.apiAtMsg(com.xxxx.agent.dto.AtMsgsSaveRequestDto)
: Read "application/json;charset=UTF-8" to [AtMsgs { message:한글한글, phone_number:string, template_code:string, reserved_date:string}]
: 한글한글
: API At Msg : {"msg":"한글한글","phoneNumber":"string","templateCode":"string","reservedDate":"string"}
```

Open API 에서는 잘 나왔다
> 그러면 문제는 클라이언트에서 API 서버로 post 요청부터 서버에서 post함수 처리 사이에서 깨진다는 것을 알 수 있었다.

어디서 이상한지 범위를 축소시켰다!

#### 문제 범위 좁히기

1) Json 데이터를 DB에 insert하고 API 서버로 전송하는 컨트롤러

```java
/*AtMsgsApiController*/
@PostMapping("/api/v1/at-msgs")
public String save(@RequestBody AtMsgsSaveRequestDto requestDto){
  Gson gson=new Gson();
  String reqData=gson.toJson(requestDto);
  log.info("Request Data : "+reqData);

  String statusCode=ApiCall.post("http://localhost:8082/api/at-msg",reqData); // API 서버로 전송
  log.info("statusCode :"+statusCode);

  if(statusCode.equals("200")){
    atMsgsService.save(requestDto); // DB에 insert
  }
  return statusCode;
}
```

- 컨트롤러에서 request 데이터는 문제 없음 확인.

2020-11-21 18:24:56.339 INFO 38496 --- [nio-8080-exec-1] c.h.controller.AtMsgsApiController :
Request Data : {"msg":"한글","phoneNumber":"8210xxxxxxxx","templateCode":"00001_00009","reservedDate":"20201123182400"}


2) API 서버로 전송하는 메소드 확인

HttpClient를 사용했다. 확인 해 본 결과 여기에 문제가 있었다.
아래는 문제가 있는 post 메소드다

~~누구도 탓 할 수 없는...내가 잘 알아보지 않고 가져다 쓴..~~

```java
/*ApiCall Class*/
public static String post(String requestURL, String jsonMessage) {
  try {
    HttpClient client = HttpClientBuilder.create().build(); // HttpClient 생성
    HttpPost postRequest = new HttpPost(requestURL); //POST 메소드 URL 새성
    postRequest.setHeader("Accept", "application/json");
    postRequest.setHeader("Connection", "keep-alive");
    postRequest.setHeader("Content-Type", "application/json; charset=utf-8");

    postRequest.setEntity(new StringEntity(jsonMessage)); //json 메시지 입력
    HttpResponse response = client.execute(postRequest);

    log.info("response.getEntity() : " + response.getEntity());

    //Response 출력
    if (response.getStatusLine().getStatusCode() == 200) {
        ResponseHandler<String> handler = new BasicResponseHandler();
        String body = handler.handleResponse(response);
        log.info("Response Data(body)"+body);
        // TODO : response handler body 가 9000이면 에러 일으키기!!!
        if (body.equals("9000")){ // produce 예외 발생시 9000
            log.error("Response is error 예외 발생: " + body);
        }
        return body;
    } else {
        log.error("Response is error : " + response.getStatusLine().getStatusCode());
        return response.getStatusLine().getStatusCode()+"";
    }
  } catch (Exception e){
        System.err.println(e.toString());
        return "4000"; // http 연결 에러
  }
}
```


### 시도3. httpclient를 사용한 post 메소드 수정으로 해결

[before]
```java
postRequest.setEntity(new StringEntity(jsonMessage)); //json 메시지 입력
HttpResponse response = client.execute(postRequest);
```

[after]
```java
HttpEntity entity = new StringEntity(jsonMessage.toString(), "UTF-8");
postRequest.setEntity(entity); //json 메시지 입력
HttpResponse response = client.execute(postRequest);
```

> 참고 : StringEntity 생성자에서 charset을 설정해 줄 수 있다.
```java
    /**
     * Creates a StringEntity with the specified content and charset. The MIME type defaults
     * to "text/plain".
     *
     * @param string content to be used. Not {@code null}.
     * @param charset character set to be used. May be {@code null}, in which case the default
     *   is {@link HTTP#DEF_CONTENT_CHARSET} is assumed
     *
     * @throws IllegalArgumentException if the string parameter is null
     * @throws UnsupportedCharsetException Thrown when the named charset is not available in
     * this instance of the Java virtual machine
     */
    public StringEntity(final String string, final String charset)
            throws UnsupportedCharsetException {
        this(string, ContentType.create(ContentType.TEXT_PLAIN.getMimeType(), charset));
    }
```

```text
: POST "/api/at-msg", parameters={}
: Mapped to public java.lang.String com.xxxxx.api.controller.AtController.apiAtMsg(com.xxxxx.agent.dto.AtMsgsSaveRequestDto)
: Read "application/json;charset=utf-8" to [AtMsgs { message:황수영, phone_number:821065362547, template_code:00001_00009, reserved_date:2020112319 (truncated)...]
: 황수영
: API At Msg : {"msg":"황수영","phoneNumber":"8210xxxxxxxx","templateCode":"00001_00009","reservedDate":"20201123192800"}
```

TIL
- 문제 범위를 차근차근 좁혀나가며 해결할 것.
- 시도했는데 되지 않았던 문제는 과감히 기록하고 넘어갈 것.(기록안하면 했는지 안했는지 까먹는 내 멍청이 뇌)
- 테스트 시간을 줄일 수 있게 노력하기
