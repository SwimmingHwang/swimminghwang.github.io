---
layout:     post
title:      Effective Java 3/E 9장. 일반적인 프로그래밍 원칙 
author:     Soo-young Hwang
tags: 		JAVA
subtitle:  	아이템57. 지역변수의 범위를 최소화하라
category:   study
---


### 서론   
<!-- 회사에서 동기들이랑 자바 스터디를 만들었다.   
나는 9장. 일반적인 프로그래밍 원칙을 맡았고 노션에 정리한 내용을 여기로 옮겼다.   -->
첫번째 아이템은 <strong>아이템57. 지역변수의 범위를 최소화하라</strong> 

### 아이템57. 지역변수의 범위를 최소화하라
<blockquote>용어 체크</blockquote>

### 지역변수란?

```java
class Ex_variables{
		// 전역변수(객체변수) : 같은클래스에서 호출이 가능
		int global_int; //private이라고하지만
		// 전역변수(클래스변수) : 다른 클래스에서도 호출이 가능
		static int global_statuc_int; 
		//Q. public or not은 설정..
		void method()
		{
			//지역변수 { }안에 생성되며 { }를 벗어나면
			int local_int = 0;  
		}//method() 라는 메소드가 끝나는 시점에 바로 삭제
}
```

**선언 위치**에 따라 전역변수(Global variable)와 지역변수(Local variable)로 나눠진다.

- 참고) 전역변수
    - 객체 변수 == 인스턴스 변수
        객체 변수는 클래스 영역에서 선언되며 클래스의 객체를 생성할 때 만들어진다. 즉 객체화를 시켜서 호출해야지만 사용이 가능하다.
    - 클래스 변수 == static 변수
        객체화를 시키지 않고도 사용이 가능. 객체 변수가 객체화 시킬때마다 서로 다른 저장공간을 가지는 반면 클래스 변수는 공통적인 저장공간을 가진다.

    ```java
    class Card {

    String kind;           

    int number;

    static int width = 200;  

    static int height = 300;    

    }
    ```

    ```java

    public class Ex_variables3 { 

    public static void main(String[] args){ 

    System.out.println("Card의 너비는 :"+ Card.width); 
    System.out.println("Card의 높이는 :"+ Card.height); 

    Card c1 = new Card(); 
    c1.kind = "Heart"; 
    c1.number = 7; 

    Card c2 = new Card(); 
    c2.kind = "Spade"; 
    c2.number = 4; 

    c1.width = 250; 
    c1.height = 350; 

    } 
    }
    ```

- 추가) 클래스
    - 클래스의 요소 : 필드와 메소드 (전에 설명함)

    인스턴스(instance)멤버란 객체(instance)를 생성한 후 사용할 수 있는 `filed`와 `method` 를 말하는데, 이들을 각각 `instance filed` , `instance method` 인스턴스 필드, 인스턴스 메소드라고 부른다. 

    즉, 아래의 예시는 인스턴스 멤버들이다. 인스턴스 필드와 인스턴스 메소드는 객체에 소속되어있는 멤버이기 때문에 **객체 없이는 사용할 수 없다.** 

    ```java
    public class Car{
    	// field 
    	int gas;

    	// method
    	void setSpeed(int speed){ ... }
    }
    ```

    위의 예시에서 `gas` 필드와 `setSpeed()` 메소드는 인스턴스 멤버이기 때문에 외부 클래스에서 사용하기 위해서는 car객체(instance)를 생성하고 참조 변수에서 접근해야 한다. 

    *참고로 인스턴스 필드 gas는 객체마다 따로 존재하고 인스턴스 메소드는 객체마다 존재하지 않고 메소드 영역에 저장되고 공유됨. 

    - `static`
        - Instance에 소속된 멤버가 아니라 클래스에 소속된 멤버이기 때문에 클래스 멤버라고도 부르는 것.
        - Static filed and method는 클래스에 고정된 멤버이므로 클래스 로더가 클래스(바이트 코드)를 로딩해서 **메소드 메모리 영역에 적재**할 때 클래스별로 관리되며, 클래스의 로딩이 끝나면 바로 사용할 수 있다.

            ```java
            public class Calculator{
            	String color; //계산기 별로 색깔이 다를 수 있다.
            	static double pi = 3.141592; // 계산기에서 사용하는 파이 값은 동일하다
            }
            ```

        - 주의할 점 !
            - 객체가 없어도 실행된다는 특징 때문에, 내부에 인스턴스 필드나 인스턴스 메소드를 사용할 수 없다. (Static 필드 메소드는 사용가능)
                - 사용하고 싶으면 인스턴스 생성해서 사용해야 함.
            - `this` 키워드 사용 불가능 → 사용 불가능 (컴파일 에러)
            - `main` 메소드도 `static` 이다. 그래서 객체 생성하고 인스턴스 필드와 메소드를 사용할 수 있다.

                ```java
                public Class Car{
                	int speed;
                	void run(){}
                	//잘못코딩
                	public static void main(String[] args){
                		speed = 60; // compile error
                		run();   // compile error
                	}
                }
                ```

                ```java
                //올바르게 수정된 코드
                public static void main(String[] args){
                	Car myCar = new Car();
                	myCar.speed = 60;
                	myCar.run();'
                }
                ```

    - 클래스의 접근 제한
        - Class 선언시 public을 생략했으면 클래스는 default 접근 제한을 가진다. Default 접근 제한을 가지면 
        **같은 패키지에서 아무런 제한 없이 사용할 수 있지만 다른 패키지에서는 사용할 수 없도록 제한된다.**

            참고* 생성자를 선언하지 않으면 컴파일러에 의해 자동으로 기본 생성자가 추가되며, 기본 생성자의 접근 제한은 클래스의 접근 제한과 동일

    - 메소드의 접근제한
        - 접근 제한자 public, protected, private을 생략했다면 default 접근 제한자가 적용된다.
        Default접근 제한은 class 접근 제한과 마찬가지로 같은 패키지에서는 사용할 수 있으나, 다른 패키지에서 사용할 수 없다.
        → 패키지에 클래스가 하나일 때는 private과 동일하다.(2차시 스터디에서 언급되었던 것처럼) 하지만, 한 패키지에 클래스가 두 개 이상이 되면 달라짐.

        ```java
        package sec13.exam03_field_method_access.package1; // 패키지 동일

        public class A{
        	// field
        	public int filed1; // public 접근 제한
        	int filed2; // protected 접근 제한 
        	private int filed3; // private 접근 제한

        	// 생성자
        	public A{
        		filed1 = 1; // ㅇ
        		filed2 = 1; // ㅇ
        		filed3 = 1; // ㅇ
        	
        		method1(); // ㅇ
        		method2(); // ㅇ
        		method3(); // ㅇ
        	}
        	// method
        	public void method1(){}
        	void method2()
        	private void method3()
        ```

        ```java
        package sec13.exam03_field_method_access.package1; // 패키지 동일

        public class B{
        	// 생성자
        	public B{
        		filed1 = 1; // ㅇ
        		filed2 = 1; // ㅇ
        		filed3 = 1; // x.  -> private field 접근 불가(컴파일 에러)
        	
        		method1(); // ㅇ
        		method2(); // ㅇ
        		method3(); // x -> private 메소드 접근 불가 (컴파일 에러)
        	}
        ```

        ```java
        package sec13.exam03_field_method_access.package1.A_diff;

        public class C{
        	// 생성자
        	public C{
        		filed1 = 1; // ㅇ
        		filed2 = 1; // x -> default field 접근 불가(컴파일 에러)
        		filed3 = 1; // x -> private field 접근 불가(컴파일 에러)
        	
        		method1(); // ㅇ
        		method2(); // x -> default field 접근 불가(컴파일 에러)
        		method3(); // x -> private 메소드 접근 불가 (컴파일 에러)
        	}
        ```

## 목표

**지역 변수의 범위를 최소화**하여 **코드 가독성과 유지보수성을 높이고 오류 가능성을 낮추자.**

### 1. 가장 처음 쓰일 때 선언하기 (= 사용할 때 선언하기)

미리 선언하면? 

- 코드가 어수선해지며 가독성이 떨어짐
- 변수를 실제로 사용하는 시점에 타입과 초깃값이 기억나지 않을 수 있음.
- 지역변수의 범위 자체가 선언된 지점부터 해당 블록({}를 의미함)이 끝날 때 까지 임.
혹시나 블록 밖에 변수를 선언하면 그 블록이 끝나도 변수는 살아있고 의도하지 않은 범위에서 사용하면 큰일남. → 즉, 사용할 때 선언하지 않으면 원하지 않는 위치에서 해당 변수를 사용하게 될 수도 있고 실수를 범할 수 있음.

**→ 미리 선언하면 여러 실수를 범할 수 있으니 가장 처음 쓰일 때 선언하자.**  

### 2. 선언과 동시에 초기화하기

**초기화에 필요한 정보가 충분해 질 때까지 선언을 미뤄서라도 선언과 동시에 초기화하기.**

- `try-catch`에서는 예외

    변수를 초기화하는 표현식에서 검사 예외를 던질 가능성이 있다면 
    try 블록 안에서 초기화 할 것 (예외가 블록을 넘어 메서드에 전파되어서)

    하지만, 블록 바깥에서도 사용해야하면(초기화 못해도) try 블록 바로 앞에서 선언하기

    ```java
    **class<? extends Set <String>> cl = null;**
    try{
    	cl = ~~
    	//생략
    } catch(~~){
    }
    ```

### 3. While문보다는 for문을 쓰는 편이 낫다.

(반복 변수의 값을 반복문이 종료된 뒤에도 써야하는 상황이 아니라면)

`for` 문의 장점

1. 복붙 오류(실수)를 방지할 수 있다.
아래 예시에서 버그를 찾아보세요!!

    ```java
    Iterator<Element> i = c.iterator();
    while(i.hasNext()){
    	doSomething(i.next());
    }
    ...

    Iterator<Element> i2 = c2.iterator();
    while(i.hasNext()){
    	doSomething(i2.next());
    }
    ...
    ```

    이는 컴파일도 잘 되고 예외도 던지지 않음. 
    하지만, 두번째 `while` 문은 c2를 순회하지 않고 곧장 끝남. 
    → c2 가 비어있다고 착각할 수 있음! 

    `for` 문을 사용하면 이런 오류를 컴파일타임에 잡아줄 수 있다. 
    ( i를 찾을 수 없다 는 컴파일 오류를 냄)

    ```java
    for(Iterator<Element i = c.iterator(); i.hasNext();){
    	Element e = i.next();
    	...
    }

    // 컴파일 오류 
    for(Iterator<Element i2 = c2.iterator(); i.hasNext();){
    	Element e2 = i2.next();
    	...
    }
    ```

2. 변수 유효 범위가 `for` 문 범위와 일치하여 똑같은 이름의 변수를 여러 반복문에서도 써도 아무런 영향을 주지 않음(세련됨)
3. `for` 문은 `while` 보다 짧아서 가독성이 좋다. 
4. `for`, `for-each` 등 반복 변수( ex : i)의 범위가 반복문의 몸체, 그리고 for 키워드와 몸체 사이의 괄호 안으로 제한된다. 
5. 그 외 지역변수 범위를 최소화하는 반복문 예시

    범위가 일치하는 두 반복 변수 `i` and `n` 를 쓸때 아래처럼 하기.
    지역 변수 범위를 최소화 하면서 반복할 때 마다 다시 계산해야 하는 비용 줄일 수 있음.

    ```java
    for(int i = 0, n = expensiveComputation(); i<n; i++){
    	...// i로 뭘 함
    }
    ```

- `for` 문 TIP
    - 컬렉션 or 배열 순회하는 권장 코드

        ```java
        for(Element e : c){
        	...//e do something
        }
        ```

        반복자(iterator)를 사용해야하는 상황아니면
        ( `for-each` , `iterator` 의 `remove` 메서드를 써야하면 )전통 for문 쓰기 

        ```java
        for(Iterator<Element> i = c.iterator(); i.hasNext();){
        	Element e = i.next();
        	...//e and i does something
        }
        ```

### 4. 메서드를 작게 유지하고 한 가지 기능에 집중하기

한 메서드에서 여러 가지 기능을 처리하면 한 기능이랑만 관련된 지역변수라도 다른 기능을 수행하는 코드에서 접근할 수 있다. 이는 실수를 범할 수 있다!

→ 메서드를 기능별로 쪼개자.