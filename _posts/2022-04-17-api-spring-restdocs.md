---
layout:     post
title:      Spring REST Docs를 이용한 API 문서 자동화하기
author:     Soo-young Hwang
tags: 		API-documentation 

---

# 서론

Log API 를 개발하면서 API 문서를 작성해야 했다.    
`Swagger`는 명료한 문서를 제공하는데 적합해 보이지 않아서 컨플루언스에 직접 작성을 했었다.    
API 개발과 문서작성을 함께 진행하다보니 API End point 가 변경될 때나 코드의 변경사항이 있을 때 매번 수정을 해야하는 단점이 있어 API 문서 자동화를 알아보게 되었고, `Spring REST Docs`를 적용해봤다.

# Spring REST Docs : RESTful 서비스 문서화작업을 도와주는 라이브러리

자바 문서 자동화에는 주로 `Swagger` 와 `Spring Rest Docs` 가 사용된다.

각각의 장점과 단점은 다음과 같다.Spring Rest Docs를 적용하면서 테스트 코드를 작성을 해야했는데, 나는 처음이라 많이 헤맸다. 테스트 코드 작성에 익숙치 않다면 적용하는데 시간이 걸릴 수 있다.

|      | **Swagger** | **Spring REST Docs**  |
| :--- | :-----------| :-------------------- |
| 장점 | API를 테스트 해 볼 수 있다.  | 테스트가 성공해야 문서가 작성된다. |
|      | 적용하기 쉽다.    | 코드와 문서가 분리되어 있다.  |
| 단점  | 코드에 어노테이션을 추가해야 한다 | 적용하기 어렵다.  ((개인) 테스트 코드 작성이 익숙치 않으면 적용하는데 더 오래 걸린다) |
|      | 코드와 동기화가 안 될 수 있다.    |       |



# 적용하기

환경은 다음과 같다.

- SpringBoot 2.6.4
- Maven
- Java 8
- JUnit 4.13.2
- AsciiDoc

Spring REST docs 적용을 중점으로 작성했다. ~~테스트 코드는 사실 처음이라 다른 문서 보시길..~~

## pom.xml

- dependency
    - `spring-restdocs-mockmvc`
        - mockmvc 를 restdoc 에서 사용할 수 있게 하는 라이브러리
- build plugin
    - `asciidoctor-maven-plugin`
        - Asciidoctor 를 사용하여 AsciiDoc문서를 html(default)로 변환함
- 더 자세한 Build Configuration 은 [공식 문서](https://docs.spring.io/spring-restdocs/docs/current/reference/html5/#getting-started-build-configuration) 찾아보기

```xml
 <dependencies>
   <dependency>
     <groupId>junit</groupId>
     <artifactId>junit</artifactId>
     <scope>test</scope>
   </dependency>
   <dependency>
     <groupId>org.springframework.restdocs</groupId>
     <artifactId>spring-restdocs-mockmvc</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.asciidoctor</groupId>
        <artifactId>asciidoctor-maven-plugin</artifactId>
        <version>1.5.8</version>
        <dependencies>
          <dependency>
            <groupId>org.springframework.restdocs</groupId>
            <artifactId>spring-restdocs-asciidoctor</artifactId>
            <version>2.0.2.RELEASE</version>
          </dependency>
        </dependencies>
        <executions>
          <execution>
            <id>asciidoc-to-html</id>
             <!-- asciidoctor가 동작하는 시점 정의 -->
            <phase>prepare-package</phase>
            <goals>
              <goal>process-asciidoc</goal>
            </goals>
            <configuration>
              <backend>html5</backend>
              <sourceHighlighter>coderay</sourceHighlighter>
              <preserveDirectories>true</preserveDirectories>
              <!-- .adoc파일들을 토대로 만든 웹 문서를 저장할 위치 선택 -->
              <outputDirectory>src/main/resources/static</outputDirectory>
              <attributes>
                <java-version>${java.version}</java-version>
                <spring-restdocs-version>${spring-restdocs.version}</spring-restdocs-version>
                <basedir>${project.basedir}</basedir>
              </attributes>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>


```

## **[기초] Document 코드 작성**

test 패키지 구조는 다음과 같다

![image-20220423180745372](https://swimmingHwang.github.io/img/Gruntfile.png)


### **1. Customizing requests and responses**

Spring REST Docs는 문서화 전처리기를 제공한다. 이는 요청이나 응답을 수정할 수 있다.예를 들어, local에서 테스트를 해 실제 url은 localhost:port 일테지만  url을 지정하는 등 전처리가 가능하다.헤더 제거도 가능하다.

테스트 코드에서는 `getDocumentRequest`, `getDocumentResponse` static 메서드를 사용하여 인스턴스를 얻어 사용한다.

### **ApiDocumentUtils**

```java
package com.xxxxx.xxxx.logapi.utils;

import static org.springframework.restdocs.operation.preprocess.Preprocessors.modifyUris;
import static org.springframework.restdocs.operation.preprocess.Preprocessors.preprocessRequest;
import static org.springframework.restdocs.operation.preprocess.Preprocessors.preprocessResponse;
import static org.springframework.restdocs.operation.preprocess.Preprocessors.prettyPrint;

import org.springframework.restdocs.operation.preprocess.OperationRequestPreprocessor;
import org.springframework.restdocs.operation.preprocess.OperationResponsePreprocessor;

public interface ApiDocumentUtils {

  static OperationRequestPreprocessor getDocumentRequest() {
    return preprocessRequest(
        modifyUris()// (1) 문서상 URI 변경
            .scheme("https")
            .host("xxxxx.xxxxxxx.com")
            .port(37000),
        prettyPrint());// (2) request 예쁘게 출력
  }

  static OperationResponsePreprocessor getDocumentResponse() {
    return preprocessResponse(prettyPrint());// (3) response 예쁘게 출력  }
  }
}
```

### **2. 테스트 코드 작성 - 예시) 인증 토큰 발급**

- andDo 뒤에 문서화에 사용되는 항목들 작성해 주면 된다.
- 요청 필드, 응답 필드를 작성할 수 있으며, 더 많은 기능은 [응용] 부분에서 자세히 다룸.
    - 타입 지정 가능
    - description 작성 가능
    - 필수값 여부 작성 가능

```java
@Test
public void 토큰발급() throws Exception {
// given
	TokenRequest tokenRequest = new TokenRequest();
	tokenRequest.setUsername("username");
	tokenRequest.setPassword("password");

// when
	ResultActions result = this.mockMvc
			.perform(post("/v1/token")
					.contentType(MediaType.APPLICATION_JSON)
					.content(mapper.writeValueAsString(tokenRequest)));

// then
	result.andExpect(status().isOk())
			.andDo(document("token-api",// 테스트 수행 시 token-api 폴더 하위에 문서가 작성된다
					getDocumentRequest(),
					getDocumentResponse(),
					requestFields(
							fieldWithPath("username").type(JsonFieldType.STRING).description("아이디"),
							fieldWithPath("password").type(JsonFieldType.STRING).description("패스워드")
					),
					responseFields(
							beneathPath("data").withSubsectionId("data"),
								fieldWithPath("username").type(JsonFieldType.STRING).description("아이디"),
								fieldWithPath("tokenType").type(JsonFieldType.STRING).description("토큰 타입"),
								fieldWithPath("accessToken").type(JsonFieldType.STRING).description("Access Token")
					)))
			.andDo(print())
			;
}

```

### **3. 테스트 실행**

해당 테스트 실행 시  `target/generated-snippets`  에 adoc 파일이 생성되는 것을 볼 수 있다.

![image-20220423180927199](https://swimmingHwang.github.io/img/image-20220423180927199.png)

adoc 파일은 이렇게 생겼고,

[예시 http-request.adoc]

```text
[source,http,options="nowrap"]
----
POST /v1/token HTTP/1.1
Content-Type: application/json;charset=UTF-8
Content-Length: 56
Host: xxxxx-api.xxxxx.com:37000

{
  "username" : "username",
  "password" : "password"
}
----

```

[예시 response-body.adoc]

```text
[source,options="nowrap"]
----
{
  "code" : "0000",
  "message" : "SUCCESS",
  "data" : {
    "accessToken" : "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJ1c2VybmFtZSIsImlhdCI6MTY1MDIyMTI0NSwiZXhwIjoxNjU3OTk3MjQ1fQ.Zu2M33AvK88QHQHzxe0n6CKaz5dYXskJxVWvF_9-LB1p9FGIX2pdUu4l4_bH8FJu_kbDqYH9CZ_XBWzPV1G9DA",
    "tokenType" : "Bearer",
    "username" : "username"
  }
}
----

```

### **4. 전체 문서 작성**

이렇게 생성된 snippet 들을 모아 adoc 문서를 작성할 수 있다.아래는 토큰 발급 문서를 일부 발췌했다.

```text
== 1. 토큰 발급

`/v1/token`

- 사이트에 등록된 상위 그룹 계정으로 로그 API 호출에 필요한 토큰을 발급받습니다.
- 발급받은 토큰을 log 조회 api 호출 시 헤더에 입력해야 합니다.
- 해당 API 를 사용하기 위해서는 사이트에서 상위그룹 생성이 필요하며 사업부로 그룹 생성 요청이 필요합니다.
- 토큰 유효기간은 3개월 입니다. 토큰 발급 후 3개월이 지난 경우 만료되기 때문에 재발급이 필요합니다.

=== Request

operation::token-api[snippets='request-fields,request-body,curl-request']

=== Response

operation::token-api[snippets='response-fields-data,response-body,http-response']


```

### **5. 빌드**

빌드까지 수행하면, 테스트 코드로 작성된 snippet들이 하나의 Html 문서로 생성된 것을 확인할 수 있다.

이에 대한 결과물은 이렇다.

![image-20220417-191743](https://swimmingHwang.github.io/img/image-20220417-191743.png)

여기까지가 기본적으로 적용 및 결과물이다.

더 나아가 필수값 여부 표현, 입력 포맷 표현, 공통 응답 및 공통 응답 코드를 자동화 할 수 있다.

## [응용] **Document 코드 작성**

Request 필드에 필수값 여부에 대해 표현을 하는 등 입력 포맷에 대해 작성할 필요가 있다. 간단한 설명과 예시를 작성했다. 이를 참고하면 쉽게 작성할 수 있다.

- 필수값 여부
- 입력 포맷
- 공통 포맷
- 계층 구조 응답

### **1. 필수값 여부**

필수값 여부를 표현하려면 테스트 코드에서 spring REST Docs에서 제공하는 optional 메서드를 사용할 수 있다. Request 필드 작성 시에 .optional() 을 붙여 선택 항목임을 알려줄 수 있고 snippet 을 customizing 하여 optional이 false일 때 필수값임을 표현할 수 있다.

- snippet customizing
    - `src/test/resources/org/springframework/restdocs/templates` `request-fields.snippet` 파일을 추가

[reqeust-fields.snippet]

```text
|===
|필드명|타입|필수여부|양식|설명

{{#fields}}
|{{#tableCellContent}}`+{{path}}+`{{/tableCellContent}}
|{{#tableCellContent}}`+{{type}}+`{{/tableCellContent}}
|{{#tableCellContent}}{{^optional}}true{{/optional}}{{/tableCellContent}}
|{{#tableCellContent}}{{#format}}{{.}}{{/format}}{{/tableCellContent}}
|{{#tableCellContent}}{{description}}{{/tableCellContent}}

{{/fields}}
|===

```

스니펫은 mustache 문법을 사용한다. 이런 식으로 커스텀 할 수 있다. 타이틀도 추가할 수 있고 표의 헤더 항목도 작성할 수 있다.

mustache 문법 중 `}}` 은 param이 비어있거나 false 일때 작동된다. (기타 mustache 문법은 다른 문서 참고..)

### **예시**

- 필드 4개 모두 선택 항목이어서 optional 을 붙여주면 된다. 필수 여부에 공백임(false)을 볼 수 있다.
    - ~~사실 false 로 표현하고 싶은데 시간상 못했음..~~

```java
1requestFields(
2		fieldWithPath("startDate").type(JsonFieldType.STRING).attributes(getDateFormat()).description("로그 조회 시작 날짜").optional(),
3		fieldWithPath("endDate").type(JsonFieldType.STRING).attributes(getDateFormat()).description("로그 조회 끝 날짜").optional(),
4		fieldWithPath("pageNum").type(JsonFieldType.NUMBER).description("페이지 번호").optional(),
5		fieldWithPath("pageSize").type(JsonFieldType.NUMBER).description("조회 건수").optional()
6),
```



![image-20220417-193942](https://swimmingHwang.github.io/img/image-20220417-193942.png)

### **2. 입력 포맷**

API spec 상 날짜 형식을 입력받아야 한다. 날짜의 경우 String 으로 입력받고 날짜 형식은 다양할 수 있어 문서에 꼭 표기를 해 줘야 했다. 입력 포맷을 지정해 주는 것도 간단하다. `attributes()` 메소드를 이용해 추가 속성을 넣어줄 수 있다.  attributes에 `key("format").value("yyyyMMddHHmmss")`  를 각각 넣어줄 수 있지만 자주 사용하므로 static 메서드로 작성해 준다.

```java
public interface DocumentFormatGenerator {
  static Attributes.Attribute getDateFormat() {
    return key("format").value("yyyyMMddHHmmss");
  }
}
```

코드 및 결과물은 위의 예시와 중복되어 패스

### **3. 공통 포맷**

공통 포맷에는 공통 응답과 공통 응답 코드가 있다. 작성 방법은 다음과 같다.

1. 공통 응답 코드를 return 해 주는 test controller 작성
2. test controller 테스트 코드 작성
    - 공통 응답
        - `customResponseFields("custom-response", ...`
            - custom-response 라는 이름으로 스니펫 생성
    - 공통 응답 코드

**[공통 응답 코드]**

```java
public enum ResultCode implements EnumType {

  SUCCESS("0000", "요청이 성공하였습니다."),
  AUTHENTICATION_FAILED("2000", "인증에 실패하였습니다."),
  BAD_REQUEST("4000", "요청 파라미터가 잘못되었습니다."),
  NOT_FOUND("4040", "리소스를 찾지 못했습니다."),
  SERVER_FILE_ERROR("5001", "서버 에러입니다."),
  ;

  private final String resultCode;
  private final String message;

  ResultCode(String resultCode, String message) {
    this.resultCode = resultCode;this.message = message;
  }
  ... 생략
}
```

**[공통 응답 코드 controller]**

```java

@RestController
public class DocumentController {

  @GetMapping("/docs")
  public ApiResult<DocsCode> findAll() {

    Map<String, String> apiResponseCodes = getDocs(ResultCode.values());

    return ApiResult.succeed(
        DocsCode.builder()
            .responseCode(apiResponseCodes)
            .build()
    );
  }

  private Map<String, String> getDocs(EnumType[] enumTypes) {
    return Arrays.stream(enumTypes)
        .collect(Collectors.toMap(EnumType::getId, EnumType::getText));
  }
}


```

**[테스트 코드]**

```java
@Test
public void commons() throws Exception {

//when
	ResultActions result = this.mockMvc.perform(
			get("/docs")
					.accept(MediaType.APPLICATION_JSON)

	);

	MvcResult mvcResult = result.andReturn();
	ApiResult<DocsCode> response = mapper.readValue(mvcResult.getResponse().getContentAsString(),
				new TypeReference<ApiResult<DocsCode>>() {});

	DocsCode resultCodeDocs = response.getData();


//then
	result.andExpect(status().isOk())
			.andDo(document("common",
					customResponseFields("custom-response", null,
							attributes(key("title").value("공통응답")),
							fieldWithPath("code").type(JsonFieldType.STRING).description("결과코드"),
							fieldWithPath("message").type(JsonFieldType.STRING).description("결과메시지"),
							subsectionWithPath("data").description("데이터")
					),
					customResponseCodeFields("custom-response-code", beneathPath("data.responseCode").withSubsectionId("responseCode"),
							attributes(key("title").value("공통 응답코드")),
							enumConvertFieldDescriptor(resultCodeDocs.getResponseCode())
					)
			));
}

private static FieldDescriptor[] enumConvertFieldDescriptor(Map<String, String> enumValues) {

	return enumValues.entrySet().stream()
			.map(x -> fieldWithPath(x.getKey()).description(x.getValue()))
			.toArray(FieldDescriptor[]::new);
}


public static CustomResponseFieldsSnippet customResponseFields(String type,
		PayloadSubsectionExtractor<?> subsectionExtractor,
		Map<String, Object> attributes, FieldDescriptor... descriptors) {
	return new CustomResponseFieldsSnippet(type, subsectionExtractor, Arrays.asList(descriptors), attributes
			, true);
}



```

### **4. 계층 구조 응답**

pagination 처리가 된 응답과 같이 계층 구조의 응답을 단순하게 표현해야 했다.

1. benethPath() 메서드를 이용해 data.data 하위 항목만 reponse field로 작성하기
    - benethPath를 하지 않으면 data.service, data.account_id 이런 식으로 표현된다
2. pagination 응답 스니펫 따로 생성
    - pagination 응답 스니펫을 따로 생성해서 명확하게 보여주고자 했다.

```java
logApiResultAction
		.andExpect(MockMvcResultMatchers
				.content()
				.string(mapper.writeValueAsString(ApiResult.succeed(logResponse))))
		.andDo(document("pips-access-log-api",
				getDocumentRequest(),
				getDocumentResponse(),
				requestHeaders(
						headerWithName("Authorization").description("Bearer auth credentials")
				),
				requestFields(
						fieldWithPath("startDate").type(JsonFieldType.STRING).attributes(getDateFormat()).description("로그 조회 시작 날짜").optional(),
						fieldWithPath("endDate").type(JsonFieldType.STRING).attributes(getDateFormat()).description("로그 조회 끝 날짜").optional(),
						fieldWithPath("pageNum").type(JsonFieldType.NUMBER).description("페이지 번호").optional(),
						fieldWithPath("pageSize").type(JsonFieldType.NUMBER).description("조회 건수").optional()
				),
				responseFields(
						beneathPath("data.data"),//(1)
							fieldWithPath("id").type(JsonFieldType.STRING).description("Unique Id").optional(),
							fieldWithPath("service_name").type(JsonFieldType.STRING).description("개인정보 데이터에 접근된 시스템 명").optional(),
							fieldWithPath("account_id").type(JsonFieldType.STRING).description("개인정보 데이터에 접근한 계정ID").optional(),
							fieldWithPath("access_time").type(JsonFieldType.STRING).description("개인정보 데이터에 접근한 시간 (ISO 8601 String)").optional(),
							fieldWithPath("access_menu").type(JsonFieldType.STRING).description("사용자(직원)이 개인정보에 접근할 때 사용한 시스템의 메뉴").optional(),
							fieldWithPath("location").type(JsonFieldType.STRING).description("IP Location").optional(),
							fieldWithPath("activity_category").type(JsonFieldType.STRING).description("개인정보 데이터 취급 activity").optional(),
							fieldWithPath("access_data").type(JsonFieldType.STRING).description("접근한 개인정보 데이터의 종류").optional(),
							fieldWithPath("data_subject").type(JsonFieldType.STRING).description("정보주체의 정보").optional(),
							fieldWithPath("bulk_query").type(JsonFieldType.STRING).description("다수의 정보주체의 정보에 접근했을 시 쿼리").optional()
				),
//(2)
				customResponseFields("response", beneathPath("data").withSubsectionId("pagination"),
						attributes(key("title").value("페이지네이션 응답코드")),
						fieldWithPath("hasNext").type(JsonFieldType.BOOLEAN).description("다음 페이지 여부"),
						fieldWithPath("data").type(JsonFieldType.ARRAY).description("로그 정보")
				)
		))
		.andReturn();
```

## 최종 결과물

![image-20220417-185120](https://swimmingHwang.github.io/img/image-20220417-185120.png)

## 마무리

처음에 테스트 코드를 작성해야 해서 많이 버벅거렸는데 한 번 작성해 두니 너무 편했다.   
(강제로 테스트 코드 짜봐서 도움되기도 했다..) 우리 팀에서 Swagger만 쓰긴하지만, 다른 REST API 를 만들고 외부에 배포해야 할 때, 사용하면 좋을 것 같아 공유한다.   
그리고 Spring REST Docs 작성하는 방법도 더 다양하니 프로젝트 특성에 맞게 활용하여 쓰면 될 것 같다.

### **참고**

- [Spring REST Docs](https://docs.spring.io/spring-restdocs/docs/current/reference/html5/)
  
- [Spring Rest Docs 적용 - 우아한형제들 기술블로그](https://techblog.woowahan.com/2597/)
  
- [Spring REST Docs에 날개를… (feat: Popup) - 우아한형제들 기술블로그](https://techblog.woowahan.com/2678/)