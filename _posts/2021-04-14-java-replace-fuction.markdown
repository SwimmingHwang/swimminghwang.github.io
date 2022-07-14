---
layout:     post
title:      Java str.replaceAll과 str.replace의 차이점
author:     Soo-young Hwang
tags: 		JAVA
subtitle:  	
category:   study
---

### `str.replaceAll` 과 `str.replace` 의 차이점

#### 결론

`str.replaceAll(정규식, 바꾸려는 문자열)`

```java
public String replaceAll(String regex, String replacement)
```

`str.replace(바뀌게 되는 문자열 target, 바꾸려는 문자열)`

```java
public String replace(CharSequence target, CharSequence replacement)
```



예제

```java
package com.swimminghwang.effectivejava;

public class replaceFuncDemo {

  public static void main(String[] args) {
    String str = "황수영 수영 김수 이수";

    str = str.replace("수", "").replace("영","");
    System.out.println(str); // 황  김 이

    str = "황수영 수영 김수 이수";
    str = str.replaceAll("[수영]",""); // "수", "영" 을 제거해줌 
    System.out.println(str); // 황  김 이
    
    str = "황수영 수영 김수 이수";
    str = str.replaceAll("[수 영]",""); // "수" "영" " " 제거
    System.out.println(str); // 황김이

  }
}
```



#### 추가

String은 불변(Immutable)클래스이기 때문에 값이 바뀌거나 ex."황수" -> "이수" replace 함수를 적용하면 변수가 갖는 참조 값(주소)에 있는 데이터 변하는 것이 아니라 새로 "이수"라는 데이터를 가지고 있는 참조 값(주소)가 생성되며 참조하는 주소가 바뀌는 것이다. 
즉, 불변성이라는 특징 때문에 replace 함수를 적용했을 때 새로운 String 객체를 반환한다. 그래서 str에 다시 할당을 해 줘야한다. 

또한 이러한 경우 `StringBuilder` 를 써서 객체가 불필요하게 생성되는 것을 막을 수 있으며 동일하게 replace, replaceAll 함수를 제공한다.
(불변클래스인 이유 : https://readystory.tistory.com/139)