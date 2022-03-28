---
published: ture
image: /img
layout: post
title: Spring Framework STS로 스프링 MVC 프레임워크 프로젝트 만들기
tags: [java, spring framework]
math: true
date: 2022-03-28 00:30
---

- <h1>스프링 부트</h1>
  
스프링 부트의 경우 프로젝트 생성이 매우 쉬운 편이다.

https://start.spring.io/ 에 들어가면 spring initializr에서 프로젝트 설정을 할 수 있고 프로젝트 이름, 패키징 방식, 자바 버전, 여러 의존성 등 여러가지 설정을 추가 한 후에 프로젝트 파일을 다운 받을 수 있다.

<center><img src="/img/make-spring-project/spring-boot-initializr.png" width="80%" height="80%"></center>
<br><br>

스프링 부트에 관해서는 다른 글로 다시 정리해 보겠다.

<br><br>

- <h1>스프링 프레임워크</h1>

스프링 프레임워크는 자체 개발도구 IDE를 제공한다. 이는 자바 IDE로 많이 알려진 Eclipse를 기반으로 하여 Spring Tools이라는 플러그인을 설치해 배포하는 방식이다.

Spring Tools를 줄여 STS라 부르며 지금은 버전 4까지 나온 상황이다. 다만 버전 4는 스프링 부트 개발자들을 위해 만들어진 IDE로 스프링 MVC 프레임워크를 사용하려면 내부에서 STS 3 확장 플러그인을 설치해야 한다.

https://spring.io/tools

위 링크에서 STS 최신버전을 다운 받을 수 있다. Eclipse 기반 IDE이므로 자바가 미리 설치되어 있어야 한다.

Eclipse 뿐만 아니라 VScode도 지원하는데 여기는 STS3 플러그인이 없어 STS로 프로젝트를 생성하고 VScode로 편집을 진행한다.

우선 IDE를 다운로드 하고 원하는 위치에 STS4라는 폴더를 만든후 IDE jar파일을 폴더에 넣어준다.

<center><img src="/img/make-spring-project/sts-folder1.png" width="80%" height="80%"></center>

jar파일을 더블클릭 하면 자동으로 압축을 풀고 디렉터리에 STS 폴더를 생성후 IDE 실행에 필요한 파일들을 넣는다. 이후 이 폴더에 들어가면

<center><img src="/img/make-spring-project/sts-folder2.png" width="80%" height="80%"></center>

SpringToolSuite4.exe가 있다. 이 프로그램이 STS IDE이며 이를 더블클릭해 실행시킨다.
그러면 로딩된 후 워크스페이스를 설정하는 창이 뜨고 이를 자기가 개발할 폴더로 지정해준다.
로딩이 완료되면 익숙한 Eclipse와 비슷한 화면이 뜬다.

<center><img src="/img/make-spring-project/sts4-main.png" width="80%" height="80%"></center>

여기서 spring boot는 Create new Spring starter Project 항목을 통해 프로젝트를 만들 수 있지만, Spring mvc 프로젝트는 항목이 보이지 않는다. 이제 STS3 플러그인을 설치해보자.

Help -> marketplace에 들어간 후 STS3을 검색한다.

그럼

<center><img src="/img/make-spring-project/marketplace-sts3.png" width="80%" height="80%"></center>

이렇데 뜨는데 우리가 필요한 건 중간에 Spring Tools 3 add-On for Spring Tools4라는 것이다. Install 버튼을 통해 설치한다.

중간에 라이센스에 동의하는지 확인창이 뜨는데 전부 agreement를 선택해 마무리 버튼을 누른다.

그러면 잠시후에 STS에서 restart 버튼이 뜰것이다. 다시 시작한다.

<br><br>

다시 시작한 STS에서 좌측 상단의 File -> New -> Other 을 선택하면 다음과 같은 화면이 뜬다.

<center><img src="/img/make-spring-project/sts4-in-sts3-addOn.png" width="80%" height="80%"></center>

중간에 보면 Spring Legacy Project라는 항목이 있는데 이것이 Spring MVC 프로젝트이다.

이것을 클릭하고 프로젝트 생성으로 들어가면 프로젝트 이름과 상세설정을 볼수 있다.

<center><img src="/img/make-spring-project/sts4-select-spring-mvc.png" width="80%" height="80%"></center>

맨 아래쪽의 Spring MVC Project를 클릭하고 프로젝트 생성을 누르면, maven에서 자동으로 필요한 의존성 파일들을 다운받고 프로젝트 빌드를 한다.

<br><br>
필자의 환경은 OpenJDK 11 LTS를 사용했고 현 시점 STS에서 권장하는 버전으로 17에서 테스트 했을 때는 버전 오류가 발생해 다운그레이드 했다. 프로젝트 이름은 com.test.test로 설정했다

<br>

다른 설명에 앞서 톰켓을 알맞는 버전으로 다운받아 STS4 폴더에 같이 넣어준다. 이후 STS4에서 File -> New -> Other 의 Server를 클릭해 다운로드한 톰켓의 버전에 맞에 셋팅을 해줘야 한다.

그 다음, Window -> preferences -> JAVA -> Installed JREs 에서 설치한 자바 jdk를 로드 해준다.

이후 General -> Workspace에서 인코딩 설정을 UTF-8로 바꿔준다. 그리고 General -> Context Type에서 text를 클릭한 후 인코딩 항목에 utf-8을 입력후 적용한다. 이후 WEB -> html, css 에서 인코딩 설정을 동일하게 utf-8로 바꿔준다.

인코딩으로 생기는 문제를 최소한으로 하기 위해 미리 설정을 바꿔주겠다.

<br>

다음은 Spring MVC의 기본 파일 구성이다.

<center><img src="/img/make-spring-project/spring-mvc-structure.png" width="80%" height="80%"></center>

여기서 중점으로 보는 부분들은 src/main/java 아래에 있는 자바 코드와 src/main/webapp 하위 디렉터리 폴더와 파일들, pom.xml이 될것이다.

기본적인 코드 작성은 com.test.test 에 작성하게 된다. 기본적으로 보이는 HomeController.java 의 코드는 다음과 같다.

```java
package com.test.test;

import java.text.DateFormat;
import java.util.Date;
import java.util.Locale;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

/**
 * Handles requests for the application home page.
 */
@Controller
public class HomeController {
	
	private static final Logger logger = LoggerFactory.getLogger(HomeController.class);
	
	/**
	 * Simply selects the home view to render by returning its name.
	 */
	@RequestMapping(value = "/", method = RequestMethod.GET)
	public String home(Locale locale, Model model) {
		logger.info("Welcome home! The client locale is {}.", locale);
		
		Date date = new Date();
		DateFormat dateFormat = DateFormat.getDateTimeInstance(DateFormat.LONG, DateFormat.LONG, locale);
		
		String formattedDate = dateFormat.format(date);
		
		model.addAttribute("serverTime", formattedDate );
		
		return "home";
	}
	
}
```

위와 같이 @Controller 어노테이션과 @RequestMapping 어노테이션으로 URL 요청을 처리하는 코드이다. 이러한 코드를 바탕으로 비즈니스 로직을 구성하고 사용자 요청에 따른 DB조회 처리구문을 작성하면 웹서버로 동작하게 될것이다.

<br>

src/main/webapp 아래에 resources폴더는 우리가 웹 서버를 프로그래밍 하면서 정적 링크를 사용할 경로이다. 예를 들어 이미지 파일들이 될 것이다.

<br>

WEB-INF -> spring 아래의 파일과 폴더들은 웹 서버 상에서는 접근이 불가능한 부분이다. 스프링의 기본적인 설정파일들이 담겨있다. WEB-INF 내부 파일들에 대해서 알아보자.

appServlet -> servlet-context.xml에는 다음과 같은 내용의 코드가 들어있다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans:beans xmlns="http://www.springframework.org/schema/mvc"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:beans="http://www.springframework.org/schema/beans"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/mvc https://www.springframework.org/schema/mvc/spring-mvc.xsd
		http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">

	<!-- DispatcherServlet Context: defines this servlet's request-processing infrastructure -->
	
	<!-- Enables the Spring MVC @Controller programming model -->
	<annotation-driven />

	<!-- Handles HTTP GET requests for /resources/** by efficiently serving up static resources in the ${webappRoot}/resources directory -->
	<resources mapping="/resources/**" location="/resources/" />

	<!-- Resolves views selected for rendering by @Controllers to .jsp resources in the /WEB-INF/views directory -->
	<beans:bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<beans:property name="prefix" value="/WEB-INF/views/" />
		<beans:property name="suffix" value=".jsp" />
	</beans:bean>
	
	<context:component-scan base-package="com.test.test" />
	
	
	
</beans:beans>
```

살펴보면 jsp 서블릿 처리에 관한 내용들이 보이고 MVC의 V에 해당하는 view에 해당하는 bean을 생성해 컨트롤러는 url요청에 대한 응답으로 /WEB-INF/views/에서 .jsp파일을 리턴하는 구조로 보인다.

또 js, css, img 등을 위한 /resources/**구분도 보인다.

<br>

root-context.xml에는 다음과 같은 내용이 들어있다

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd">
	
	<!-- Root Context: defines shared resources visible to all other web components -->
		
</beans>
```

bean 선언에 관한 내용 말고는 추가된 내용은 없어보인다.

그럴수 밖에 없는것이 root-context.xml는 DAO(Data Access Object), VO(Value Object)등 DB연동 관련 서비스에 관한 설정이 적용되는 부분으로 지금은 mybatis나 mariaDB 같이 외부 데이터베이스에 대해 설정을 한것이 없기 떄문이다.

<br>

view -> home.jsp는 servlet-context.xml에서 설정한것 같이, 컨트롤러에 URL 요청이 들어오면 리턴되는 jsp 파일들이 저장되는 곳이다. MVC의 V인 view에서는 jsp를 통해 렌더링 된 화면을 서블릿을 통해 반환한다.

```xml
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ page session="false" %>
<html>
<head>
	<title>Home</title>
</head>
<body>
<h1>
	Hello world!  
</h1>

<P>  The time on the server is ${serverTime}. </P>
</body>
</html>
```

<br>

이런 코드를 가지고 있으며 이 상태로 프로젝트를 돌리면 아마 화면에서 인코딩 에러가 날 것이다. 따라서 개발중 생성되는 jsp파일들의 첫번째 줄에는 

```xml
<%@ page language="java" contentType="text/html; charset=utf-8" pageEncoding="utf-8"%>
```

을 붙여 개발한다.

<br>

다음 web.xml을 살펴보자.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://JAVA.sun.com/xml/ns/javaee https://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">

	<!-- The definition of the Root Spring Container shared by all Servlets and Filters -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/spring/root-context.xml</param-value>
	</context-param>
	
	<!-- Creates the Spring Container shared by all Servlets and Filters -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<!-- Processes application requests -->
	<servlet>
		<servlet-name>appServlet</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/spring/appServlet/servlet-context.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
		
	<servlet-mapping>
		<servlet-name>appServlet</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>

</web-app>
```

코드를 살펴보면 서블릿 설정과 컨텍스트관련 경로 설정을 잡아준다. WAS에서 필요한 Servlet설정들을 명시해주는 부분이다.

<br>

프로젝트 이름이 test라 햇갈릴수 있지만.... test 키워드로 /src/test 등 폴더들은 JUnit test에 사용된 코드와 class들이 저장되는 곳이다. 

<br>

마지막으로 pom.xml을 살펴보자. Spring은 Maven의 의존성을 통해 버전관리를 하고 원하는 라이브러리를 편하게 다운받을 수 있다. 이런 Maven의 설정에 관한 코드를 pom.xml에 작성한다. 스프링 버전, 자바 버전, 기타 라이브러리 등 모든 버전과 관련된 설정파일은 전부 여기에 작성된다.

코드를 살펴보자.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/maven-v4_0_0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.test</groupId>
	<artifactId>test</artifactId>
	<name>test</name>
	<packaging>war</packaging>
	<version>1.0.0-BUILD-SNAPSHOT</version>
	<properties>
		<java-version>11</java-version>
		<org.springframework-version>5.3.17</org.springframework-version>
		<org.aspectj-version>1.6.10</org.aspectj-version>
		<org.slf4j-version>1.6.6</org.slf4j-version>
	</properties>
	<dependencies>
		<!-- Spring -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
			<version>${org.springframework-version}</version>
			<exclusions>
				<!-- Exclude Commons Logging in favor of SLF4j -->
				<exclusion>
					<groupId>commons-logging</groupId>
					<artifactId>commons-logging</artifactId>
				 </exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
			<version>${org.springframework-version}</version>
		</dependency>
				
		<!-- AspectJ -->
		<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjrt</artifactId>
			<version>${org.aspectj-version}</version>
		</dependency>	
		
		<!-- Logging -->
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-api</artifactId>
			<version>${org.slf4j-version}</version>
		</dependency>
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>jcl-over-slf4j</artifactId>
			<version>${org.slf4j-version}</version>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-log4j12</artifactId>
			<version>${org.slf4j-version}</version>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>log4j</groupId>
			<artifactId>log4j</artifactId>
			<version>1.2.15</version>
			<exclusions>
				<exclusion>
					<groupId>javax.mail</groupId>
					<artifactId>mail</artifactId>
				</exclusion>
				<exclusion>
					<groupId>javax.jms</groupId>
					<artifactId>jms</artifactId>
				</exclusion>
				<exclusion>
					<groupId>com.sun.jdmk</groupId>
					<artifactId>jmxtools</artifactId>
				</exclusion>
				<exclusion>
					<groupId>com.sun.jmx</groupId>
					<artifactId>jmxri</artifactId>
				</exclusion>
			</exclusions>
			<scope>runtime</scope>
		</dependency>

		<!-- @Inject -->
		<dependency>
			<groupId>javax.inject</groupId>
			<artifactId>javax.inject</artifactId>
			<version>1</version>
		</dependency>
				
		<!-- Servlet -->
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>servlet-api</artifactId>
			<version>2.5</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>javax.servlet.jsp</groupId>
			<artifactId>jsp-api</artifactId>
			<version>2.1</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>jstl</artifactId>
			<version>1.2</version>
		</dependency>
	
		<!-- Test -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.7</version>
			<scope>test</scope>
		</dependency>        
	</dependencies>
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-eclipse-plugin</artifactId>
                <version>2.9</version>
                <configuration>
                    <additionalProjectnatures>
                        <projectnature>org.springframework.ide.eclipse.core.springnature</projectnature>
                    </additionalProjectnatures>
                    <additionalBuildcommands>
                        <buildcommand>org.springframework.ide.eclipse.core.springbuilder</buildcommand>
                    </additionalBuildcommands>
                    <downloadSources>true</downloadSources>
                    <downloadJavadocs>true</downloadJavadocs>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.5.1</version>
                <configuration>
                    <source>1.6</source>
                    <target>1.6</target>
                    <compilerArgument>-Xlint:all</compilerArgument>
                    <showWarnings>true</showWarnings>
                    <showDeprecation>true</showDeprecation>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>1.2.1</version>
                <configuration>
                    <mainClass>org.test.int1.Main</mainClass>
                </configuration>
            </plugin>
        </plugins>
        <pluginManagement>
        	<plugins>
        		<!--This plugin's configuration is used to store Eclipse m2e settings only. It has no influence on the Maven build itself.-->
        		<plugin>
        			<groupId>org.eclipse.m2e</groupId>
        			<artifactId>lifecycle-mapping</artifactId>
        			<version>1.0.0</version>
        			<configuration>
        				<lifecycleMappingMetadata>
        					<pluginExecutions>
        						<pluginExecution>
        							<pluginExecutionFilter>
        								<groupId>
        									org.apache.maven.plugins
        								</groupId>
        								<artifactId>
        									maven-compiler-plugin
        								</artifactId>
        								<versionRange>
        									[2.5.1,)
        								</versionRange>
        								<goals>
        									<goal>testCompile</goal>
        								</goals>
        							</pluginExecutionFilter>
        							<action>
        								<ignore></ignore>
        							</action>
        						</pluginExecution>
        					</pluginExecutions>
        				</lifecycleMappingMetadata>
        			</configuration>
        		</plugin>
        	</plugins>
        </pluginManagement>
    </build>
</project>
```

pom.xml에 정의되어 있는 의존성에 대한 버전정보는 https://mvnrepository.com/ 에서 확인이 가능하고 원하는 버전을 사용할 수 있다. 다만 주의해야 할 점이 Spring 버전과 openjdk 버전을 맞춰줘야 한다. 이러한 버전관리에서 발생하는 복잡성 때문에 gradle이라는 기술이 등장해 Spring Boot에 적용되었다.

다만 개인적으로 필자는 아직도 Maven이 익숙해 종종 사용하고 있다.

<br><br>

이제 테스트 프로젝트를 실행해보자.

위의 Tomcat 설정을 진행하면서 프로젝트 모듈을 Tomcat에 임포트 하는 과정이 있을 것이다. Window -> Show View -> Other 에서 Server를 클릭하면 Server 탭이 하나 열리고 추가된 서버가 보일 것이다. 

<center><img src="/img/make-spring-project/sts4-server-window.png" width="80%" height="80%"></center>

필자는 톰켓 9.0버전을 사용한다.

Tomcat v9.0 Server at localhost 부분을 더블클릭하면 다음과 같은 화면을 볼수 있다.

<center><img src="/img/make-spring-project/sts4-server-webModules.png" width="80%" height="80%"></center>

여기서 추가된 프로젝트 웹 모듈을 확인할 수 있고, 혹시라도 추가되어 있지 않으면 Add Web Module을 클릭해 프로젝트를 추가해준다.

이후 톰켓을 실행시키면 프로젝트 기본 페이지를 볼수 있다.

<center><img src="/img/make-spring-project/spring-mvc-running.png" width="80%" height="80%"></center>