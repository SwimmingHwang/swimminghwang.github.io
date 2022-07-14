---
layout:     post
title:      Spring Boot 와 JPA - Bulk Insert 하는 법
author:     Soo-young Hwang
tags: 		JPA Spring-Boot
subtitle:  	JPA의 기본키 생성 전략, JPA에서 Bulk Insert하는 법
category:   study
---


<blockquote> Bulk Insert란? </blockquote>

DB에 레코드를 N(N>=2) 삽입 시, insert문을 N번 반복하는 것이 아닌 하나의 Insert문에 N개의 데이터를 담아서 삽입하는 것을 말한다.   
(= Bulk Insert시 `Batch` 사이즈를 정해줘야 한다. 그래서 그런지 `Batch Insert` 라는 말로 혼용해서 쓰는 듯 하다. )
아래의 예시를 보면 <strong>`Bulk Insert`</strong>가 뭔지 바로 알 수 있을 것이다.   
​
```sql
// 개별 
INSERT INTO table1 (col1, col2) VALUES (val11, val12);
INSERT INTO table1 (col1, col2) VALUES (val21, val22);
INSERT INTO table1 (col1, col2) VALUES (val31, val32);
// Bulk
INSERT INTO table1 (col1, col2) VALUES
(val11, val12),
(val21, val22),
(val31, val32);
```

현재 프로젝트에서 JPA 구현체로 `Hibernate`를 사용한다.   
`Hibernate`를 사용하면서 엔터티의 식별자 생성에 `IDENTITY` 방식을 사용했다.   
이는 `Hibernate`가 `JDBC` 수준에서 `Bulk insert`를 비활성화한다.    

(출처 : https://homoefficio.github.io/2020/01/25/Spring-Data%EC%97%90%EC%84%9C-Bulk-Insert-%EC%B5%9C%EC%A0%81%ED%99%94/)

결국 <strong>`Hibernate`에서 `IDENTITY `방식으로 식별자를 생성하면 `Bulk insert`는 사용할 수 없다.</strong>   ​
위의 블로그를 참고하니 해결 방법은 2가지다.


### 1. `Sequence`나 `Table`방식을 사용하면서 채번 부하를 낮추기
### 2. `Spring Data JPA`(구현체 : Hibernate) 대신 `Spring Data JDBC` 사용하기 

​
1번 방식이 이해가 잘 안가서 관련 내용을 더 찾아봤다.   
[참고사이트](https://gmlwjd9405.github.io/2019/08/12/primary-key-mapping.html)

<blockquote>기본키 자동 생성 전략</blockquote>

먼저 Spring에서 기본키를 매핑하고 자동 생성 전략을 적용하는 예시를 보자.   
아래는 Spring 에서의 자동 생성 전략 중 `AUTO`로 적용한다.   

```java
@Id // 기본키 매핑
@GeneratedValue(strategy = GenerationType.AUTO) // 자동 생성 전략 중 AUTO로 설정
private Long id;
```

`@Id`로 `Entity`에서 기본키를 매핑한다.   
`@GeneratedValue`를 설정해주지 않으면 데이터 삽입 시 기본키 값을 직접 할당 해야 한다. 

​

자동 생성 전략에는 네가지가 있다.   

1. IDENTITY
2. SEQENCE
3. TABLE
4. AUTO

​
1. IDENTITY

id 값을 null로 하면 DB가 알아서 AUTO_INCREMENT를 해 준다. (MySQL)
```sql
create table User(
  id varchar(255) auto_increment,
  ...
)
```

2. SEQUENCE
DB의 Sequence Object를 사용하여 DB가 숫자를 generate해주는 방식이다.    
MySQL에는 Sequence Object가 없어 Table방식을 사용하게 된다.   
추가적으로, Orable, H2 등 에는 사용가능하다.   
3. TABLE
키 생성 전용 테이블을 하나 만들어 관리를 한다.   
공용 시퀀스 테이블을 두고 모든 테이블의 id sequence를 한 테이블에서 관리한다.   
`@TableGenerator` annotation이 필요하다.   
최적화가 되어있지 않다고 한다.
4. AUTO
기본 시퀀스 전략이며  데이터베이스 벤더가 위의 3가지 방법 중 하나를 선택한다.   

​
다시 돌아와서, `Bulk Insert` 을 해 보자.

1. Sequence나 Table방식을 사용하면서 채번(sequence)  부하를 낮추기
2. Spring Data JPA(구현체 : Hibernate) 대신 Spring Data JDBC 사용하기 


1번 방식을 적용할까 고민을 했다.

[JPA-GenerationType-별-INSERT-성능-비교](https://github.com/HomoEfficio/dev-tips/blob/master/JPA-GenerationType-별-INSERT-성능-비교.md)    
[Spring Data에서 Batch Insert 최적화](https://homoefficio.github.io/2020/01/25/Spring-Data%EC%97%90%EC%84%9C-Batch-Insert-%EC%B5%9C%EC%A0%81%ED%99%94/)


위 포스트에 따르면 Sequence, table 방식을 조치 없이 사용하면 `Bulk Insert`를 쓰더라도 `IDENTITY` 방식보다 더 느리다고 한다.   


그렇다면, Sequence, Table방식을 잘 사용해서 부하를 낮추는 방안을 고민해봐야 한다.  
Spring Data JPA가 안겨주는 편리함 뒤에는 가끔 <strong>성능 손실</strong>이 숨어있다.    
이번에 알아볼 Bulk Insert도 그런 예 중 하나다. 성능 손실 문제가 발생하는 이유와 2가지 해결 방법을 알아본다.   

위의 블로그에서 분석을 잘 해뒀다. 덕분에, Spring Data JDBC 가 가장 빠르다고 결론을 내셨기에,   
이를 적용하기로 했다.(후에 Table방식으로 적용했다)

​

<blockquote>Spring Data JPA(구현체 : Hibernate) 대신 `Spring Data JDBC` 사용하기</blockquote>

프로젝트의 주제 성격상 많은 갯수의 데이터를 Insert하는 경우가 많다. 약 10만 건 이상? (물론 가정이다)    
배치 크기에 따라 다르지만 이 방식은 Hibernate Bulk Sequence 보다 대략 2~3배 정도 빠르고    
IDENTITY보다 2~25배 빠르다는 분석이 있다.   

`JdbcTemplate.BulkUpdate()` 라는 메서드를 사용하여 구현해 적용해봤다.  
Bulk Insert가 되는 것을 확인할 수 있었다. 

~~그런데, 이미 `GenerationType`이 `IDENTITY`여서 그런지 개별 INSERT가 되었다.~~   
~~->사실은 로그만 개별로 보였을 뿐 Bulk insert가 되고 있었다. 아래에 해결 과정 기술함.~~   

​

`Bulk update`메서드를 적용한 코드는 아래와 같다.

```java
package com.xxxxx.domain.msgs;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Profile;
import org.springframework.jdbc.core.BulkPreparedStatementSetter;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;

@Slf4j
@Repository
@RequiredArgsConstructor
public class AtMsgsJdbcRepositoryImpl implements AtMsgsJdbcRepository{

    private final JdbcTemplate jdbcTemplate;

    private int BulkSize = 500;

    @Override
    public void saveAll(List<AtMsgs> atMsgs) {
        int BulkCount = 0;
        List<AtMsgs> subItems = new ArrayList<>();
        for (int i = 0; i < atMsgs.size(); i++) {
            subItems.add(atMsgs.get(i));
            if ((i + 1) % BulkSize == 0) {
                BulkCount = BulkInsert(BulkSize, BulkCount, subItems);
            }
        }
        if (!subItems.isEmpty()) {
            BulkCount = BulkInsert(BulkSize, BulkCount, subItems);
        }
        log.info("BulkCount: " + BulkCount);

    }

    private int BulkInsert(int BulkSize, int BulkCount, List<AtMsgs> atMsgs) {

        jdbcTemplate.BulkUpdate("INSERT INTO imc_at (`message`, `phone_number`) VALUES (?, ?)",
                new BulkPreparedStatementSetter() {
            // TODO 직접 insert 하기 때문에 default값 재설정 필요
                    @Override
                    public void setValues(PreparedStatement ps, int i) throws SQLException {
                        ps.setString(1, atMsgs.get(i).getMsg());
                        ps.setString(2, atMsgs.get(i).getPhoneNumber());
                    }
                    @Override
                    public int getBulkSize() {
                        return atMsgs.size();
                    }
                });
        atMsgs.clear();
        BulkCount++;
        return BulkCount;
    }

}
```


### 겪은 문제들
<blockquote>Table 방식으로 genertation type변경 후 Bulk insert 안 되는 문제 해결</blockquote>

1번 방식대로 할 때 bulk insert가 되어야 하는데 안 되는 문제가 있었다.   
즉, GenerationType도 테이블로 바꾸고 properties도 다 설정 했는데 자꾸 개별 insert가 되는 문제가 있었다.  
```
Hibernate: select tbl.next_val from hibernate_sequences tbl where tbl.sequence_name=? for update
Hibernate: update hibernate_sequences set next_val=?  where next_val=? and sequence_name=?
Hibernate: select tbl.next_val from hibernate_sequences tbl where tbl.sequence_name=? for update
Hibernate: update hibernate_sequences set next_val=?  where next_val=? and sequence_name=?
Hibernate: select tbl.next_val from hibernate_sequences tbl where tbl.sequence_name=? for update
Hibernate: update hibernate_sequences set next_val=?  where next_val=? and sequence_name=?
Hibernate: select tbl.next_val from hibernate_sequences tbl where tbl.sequence_name=? for update
Hibernate: update hibernate_sequences set next_val=?  where next_val=? and sequence_name=?
Hibernate: select tbl.next_val from hibernate_sequences tbl where tbl.sequence_name=? for update
Hibernate: update hibernate_sequences set next_val=?  where next_val=? and sequence_name=?
Hibernate: select tbl.next_val from hibernate_sequences tbl where tbl.sequence_name=? for update
Hibernate: update hibernate_sequences set next_val=?  where next_val=? and sequence_name=?
Hibernate: select tbl.next_val from hibernate_sequences tbl where tbl.sequence_name=? for update
Hibernate: update hibernate_sequences set next_val=?  where next_val=? and sequence_name=?
Hibernate: select tbl.next_val from hibernate_sequences tbl where tbl.sequence_name=? for update
Hibernate: update hibernate_sequences set next_val=?  where next_val=? and sequence_name=?
Hibernate: select tbl.next_val from hibernate_sequences tbl where tbl.sequence_name=? for update
Hibernate: update hibernate_sequences set next_val=?  where next_val=? and sequence_name=?
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
```

구글링..   
(https://stackoverflow.com/questions/21530112/no-matter-what-i-cant-Bulk-mysql-insert-statements-in-hibernate)

 
No matter what, I can't Bulk MySQL INSERT statements in Hibernate   
I'm currently facing the well-known and common Hibernate insert Bulk problem. I need to save Bulkes 5 millions of rows long. I'm first trying with a much lighter payload. Since I have to insert   

stackoverflow.com
여기서 답을 찾았다.

Hibernate는 드라이버에 여러 개의 insert문을 보낸다.    
이 출력된 개별 insert문은 드라이버가 DB에 보낸 SQL이 아니라   
<strong>SQL Hibernate가 드라이버에 보낸 내용만 보여주는 것이었다.</strong>   

`profileSQL=true`    

```
spring.datasource.url=jdbc:mysql://localhost:3306/db명?serverTimezone=UTC&allowPublicKeyRetrieval=true&useSSL=false&rewriteBulkedStatements=true&profileSQL=true
rewriteBulkedStatements=true&profileSQL=true
```

이를 true로 해주면 실제로 DB에서 어떤SQL문이 실행되는지 알 수 있다.

```
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Hibernate: insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values (?, ?, ?, ?, ?, ?, ?, ?, ?)
Sun Nov 22 23:13:39 KST 2020 INFO: [QUERY] insert into imc_at (etc1, message, phone_number, priority, reserved_date, sender_key, status, template_code, id) values ('0', 'aa', '821011111111', 'N', '20201122231300', '54ef196697bda7dbc36a45a334beb83580d8ca2a', '1', 'null', 66),('0', 'aa', '821022222222', 'N', '20201122231300', '54ef196697bda7dbc36a45a334beb83580d8ca2a', '1', 'null', 67),('0', 'aa', '821033333333', 'N', '20201122231300', '54ef196697bda7dbc36a45a334beb83580d8ca2a', '1', 'null', 68),('0', 'aa', '821044444444', 'N', '20201122231300', '54ef196697bda7dbc36a45a334beb83580d8ca2a', '1', 'null', 69),('0', 'aa', '821055555555', 'N', '20201122231300', '54ef196697bda7dbc36a45a334beb83580d8ca2a', '1', 'null', 70) [Created on: Sun Nov 22 23:13:39 KST 2020, duration: 0, connection-id: 7244, statement-id: 0, resultset-id: 0,	at com.zaxxer.hikari.pool.ProxyStatement.executeBulk(ProxyStatement.java:128)]
```

개별 insert되는 줄 알았지만,,Bulk insert가 잘 되고 있었다.   
하루종일 삽질했다.   
​
Table 방식으로 배치 insert를 시도했지만 성능에 대한 문제는 더 고민해 봐야한다.  
(Spring Data JDBC가 성능이 제일 좋다고 하기에 JDBC 를 반영했다)
​

무튼 하루종일 삽질 끝에, 기본키 자동 생성 전략 네가지도 구체적으로 알 수 있었고,   
Bulk insert를 구현하고 적용해 볼 수 있었다. 