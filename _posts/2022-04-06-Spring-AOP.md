---
published: ture
image: /img
layout: post
title: Spring Framework AOP(Aspect Oriented Programming)이란? 
tags: [java, spring framework]
math: true
date: 2022-04-06 16:30
---

스프링은 코드의 반복 사용을 줄이고 효율적이며 결합도가 낮은 유연한 코드를 작성하길 원한다.

우리는 코드를 작성할때 크게 중요한 부분은 아니지만 어떤 값을 확인하는 등 여러 부분에서 중복으로 쓰이는 코드가 있을 수 있다. 예를 들면 로그인 세션이 남아 있는지, 또는 어떤 로직에서의 에러에 대한 로그를 핸들링 할때 등등 상황은 매우 다양하다.

OOP는 코드의 재활용성을 높이고 객체지향을 통해 코드 개발을 더 쉽게, 유지보수하기 편하게 하기 위해 시작되었다. 여기서 더 나아가 Spring에서는 AOP를 통해 비즈니스 로직 상에 중복적이지만 꼭 필요한 코드를 따로 묶어 외부로 분리해 메인 코드에 집중할 수 있게 해주는 기법이다.

여기서 AOP는 따로 분리된 Aspect를 모아 모듈화 하는 기법이라고 보면 편할 것이다.

<br><br>

## 위빙(Weaving)
AOP에서 공통적으로 실행되는 기능을 직접적으로 호출하지 않고 위빙이라는 작업을 통해 호출하게 된다. 이런 위빙을 사용하기 위해서는 공통적으로 쓰이는 코드가 언제, 어디서 적용할 것인지 명시해야 한다. 어디서 적용할 것인지에 대한 설정을 Pointcut이라고 하고, 언제 적용할 것인지에 대해 Advice라 한다. 이 Pointcut과 Advice의 조합을 Aspect라고 한다.

<br><br>

우선 진행하기 전에 필요한 사전 설정을 마무리 해보자.

```xml
    <properties>
		<java-version>11</java-version>
		<org.springframework-version>5.3.17</org.springframework-version>
		<org.aspectj-version>1.9.9.1</org.aspectj-version>
		<org.slf4j-version>1.6.6</org.slf4j-version>
	</properties>

        <dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjrt</artifactId>
			<version>${org.aspectj-version}</version>
		</dependency>
		<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjweaver</artifactId>
			<version>${org.aspectj-version}</version>
		</dependency>
```

위와 같이 aop사용을 위해 maven버전을 맞춰준다. 이후 servlet-context.xml에 다음과 같은 구문을 추가한다.

```xml
xmlns:aop="http://www.springframework.org/schema/aop"

xsi:schemaLocation 에 http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.3.xsd" 추가
```

<br><br>

## Pointcut
어디에 공통관심 코드를 적용할 것인지에 대한 설정을 진행한다. 하나의 @Aspect 안에 여러개의 포인트 컷 설정이 가능하다.

```java
@Pointcut("execution(com.test.test..)")
private void all() {}
```

위와 같이 포인트 컷이 적용될 위치를 지정해 주는데 지시자로 주로 많이 사용하는 excution 타입에 대해 간략하게 적어본다.

1. 리턴타입 지정
    말 그대로 지정한 리턴타입에 적용한다.

    ```java
    *       //모든 리턴타입 혀용
    void    //리턴타입이 void인 메소드
    !void   //리턴타입이 void가 아닌 메소드
    ```

2. 패키지 지정
   Aspect 코드를 적용할 패키지를 지정한다.

   ```java
    com.test.domain     //com.test.domain 패키지만 선택
    com.test.domain..   //com.test.domain으로 시작하는 패키지 선택
   ```

3. 클래스 지정
    ```java
    userVO      //userVO 클래스만 선택
    *VO         //VO로 끝나는 클래스
    testClass   //클래스 이름 뒤에 +가 붙으면 해장 클래스로 부터 파생된 모든 자식 클래스까지 선택, 인터페이스 이름 뒤에 +가 붙으면 해당 인터페이스를 구현한 모든 클래스 선택
    ```

4. 메소드 지정
   ```java
    *(..)       //모든 메소드 선택
    update(..) //메소드명이 update로 시작하는 모든 메소드
   ```

조금 다르게 응용하자면 Advice어노테이션에 적용도 가능하다
예를 들면 @Around(value="execution(* com.test.test.*.*(..))") 형식이다.

<br><br>

## 5가지의 Advice(언제)
Advice는 다섯가지 상황으로 적용할 수 있다. 각각 어노테이션으로 적용이 가능하며 그 목록은 다음과 같다.

```java
@Before             //메서드 실행 전
@After              //메서드 실행 후
@AfterRunning       //실행 뒤 반환 후
@AfterThrowing      //예외가 던져지는 시점
@Around             //메서드 호출 전
```

Advice를 작성할떄 반드시 하나 이상의 Pointcut을 명시해야 한다.

여기서 당면한 상황에 맞게 사용하면 될것 같다. 기본적인 예를 들어보면 다음과 같은 코드로 사용이 가능하다.

### AopTestAspect

```java
@Component
@Aspect
public class AopTestAspect {
	
	Logger logger = LoggerFactory.getLogger(AopTestAspect.class);
	
	@Around("all()")
	public Object AopTest(ProceedingJoinPoint joinPoint) throws Throwable{
		
		Date date = new Date();
		date.getTime();
		Object ret = joinPoint.proceed();
		logger.info("AOP start - "+date.toString());
		
		return ret;
	}
	
	@Pointcut("@annotation(AopTest)")
	private void all() {}

}
```

### AopTest

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AopTest {}
```

### HomeContoller (AOP 어노테이션 적용 부분)

```java
	@AopTest
	@RequestMapping(value = "/aop", method = RequestMethod.GET)
	public String aop() {
		
		logger.info("aop controller start");
		
		return "home";
	}
 ```

 여기서 AopTestAspect 부분은 이렇게 사용이 가능하다.

### AopTestAspect

```java
@Component
@Aspect
public class AopTestAspect {
	
	Logger logger = LoggerFactory.getLogger(AopTestAspect.class);
	
	@Around("@annotation(AopTest)")
	public Object AopTest(ProceedingJoinPoint joinPoint) throws Throwable{
		
		Date date = new Date();
		date.getTime();
		Object ret = joinPoint.proceed();
		logger.info("AOP start - "+date.toString());
		
		return ret;
	}

}
```

 위 코드는 @AopTest 어노테이션이 쓰인 곳에 시간을 로그로 찍는 AOP를 작성했다. 아래는 AOP 사용의 또다른 예제이다.

### AopTestAspect

```java
@Component
@Aspect
public class AopTestAspect {
	
	private static final Logger logger = LoggerFactory.getLogger(AopTestAspect.class);
	
	@Around("all()")
	public Object AopTest(ProceedingJoinPoint joinPoint) throws Throwable{
		
		Date date = new Date();
		date.getTime();
		Object ret = joinPoint.proceed();
		logger.info("AOP start - "+date.toString());
		
		return ret;
	}
	
	@Pointcut("execution(* com.test.test.HomeController.*(..))")
	private void all() {}
}
```

위 코드는 com.test.test.HomeController 아래의 모든 메소드에 AOP를 적용한 코드이다. execution을 설명하자면 앞의 *는 리턴타입 설정 부분인데 이는 모든 리턴타입을 의미한다. 띄어쓰기 후 com.test.test.HomeController.*(..)은 com.test.test.Homcontroller의 위치에 모든 메소드에 AOP를 적용했다는 의미가 된다.

위 코드는 역시 다음과 같이 간략하게 작성이 가능하다.

### AopTestAspect

```java
@Component
@Aspect
public class AopTestAspect {
	
	private static final Logger logger = LoggerFactory.getLogger(AopTestAspect.class);
	
	@Around(value="execution(* com.test.test.HomeController.*(..))")
	public Object AopTest(ProceedingJoinPoint joinPoint) throws Throwable{
		
		Date date = new Date();
		date.getTime();
		Object ret = joinPoint.proceed();
		logger.info("AOP start - "+date.toString());
		
		return ret;
    }
}
```