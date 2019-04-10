## Spring Boot 란

## Microservice 란


## Microservice 의 구조
(그림으로 보는 구조)

Spring Boot를 이용해 만드는 마이크로서비스의 구조와 큰 역할은 다음과 같다.


- boot : 서비스의 전체적 설정 등의 역할을 한다.
- domain : 도메인 객체, 비즈니스 로직 등의 핵심 역할을 담당한다.
	- entity : 
	- event : 
	- lifecycle : 
	- logic : 
	- proxy : 
	- spec : 
	- store : 
- service : 외부와의 통신(url) 등을 담당함.
	- bind : 
	- lifecycle : 
	- listen : 
	- logic : 
	- rest : 
- store-jpa : DB와 JPA의 연결을 담당.
	- lifecycle : 
	- store : 
		- jpo : 
		- repository : 

## 프로젝트 전 환경
내가 사용한 환경은 다음과 같다.
- Spring Boot : 2.0.4
(boot에 있는 pom.xml에밖에 boot 버전 관련 표기된게 없어서 이거 같긴 한데 정확히 맞는지는 모르겠다.)     
~~~
	<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
    </parent>
~~~
https://eglowc.tistory.com/38 에 따르면 spring-boot-starter-parent 에는 다음과 같은 특징이 있다고 한다.
1. 기본 컴파일 레벨을 Java 1.6 으로 지정
2. UTF-8 소스 인코딩
3. spring-boot-starter-dependencies를 상속하여 <version>태그를 생략하고 의존성을 관리
4. resource filtering
5. exec plugin, surefire, Git commit ID, shade

4번과 5번이 무슨 말일까?

- STS3
- Java 1.8
- Maven

## 예제(Item)
역시 실제 코드로 알아보는 게 가장 좋다고 생각한다.

예제로 Item을 서비스하는 프로젝트를 만들어보자.

전체 구조에서 Item 프로젝트에 들어있는 파일은 다음과 같다.
- boot :
- domain :
	- entity : 
	- event : 
	- lifecycle : 
	- logic : 
	- proxy : 
	- spec : 
	- store : 
- service
	- bind : 
	- lifecycle : 
	- listen : 
	- logic : 
	- rest : 
- store-jpa
	- lifecycle : 
	- store : 
		- jpo : 
		- repository : 
---
### 1. domain

#### entity
우선 엔티티 객체를 보며 무슨 역할을 하는지 알아보자.

Item.java
~~~
 private String id;
 private String name;
 private int buyingPrice;
 private int price;
 
 private IdName seller;
~~~

물건의 id, 이름, 구입가,판매가, 그리고 파는 사람이 누군지에 대한 정보가 들어있다.

#### spec

#### logic

#### store
---
### 2. service

---
### 3. store-jpa 

## 실제 구동
