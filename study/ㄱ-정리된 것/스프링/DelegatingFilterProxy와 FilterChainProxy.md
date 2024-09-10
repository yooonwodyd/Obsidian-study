> DelegatingFilterProxy와 FilterChainProxy가 무엇인지 설명하기 위해 다음의 작업을 거친다.
> 
> 1. 서블릿 필터
> 2. 서블렛 컨테이너
> 3. DelegatingFilterProxy
> 4. FilterChainProxy

## 서블릿 필터

서블릿 필터는 ServletDispatcher에 도달하기 전 데이터를 알맞게 가공하고, 공통의 관심사를 처리하는 역할을 한다.  
(_java Servlet Filter is used to intercept the client request and do some pre-processing._)

이를 통해 서블릿에서 인증(Authentication), 인가(Authorization), 비밀보장(Confidentiality), 데이터 무결성(Data Integrity)과 같은 보안 처리가 가능하다. 따라서 우리가 별도의 보안 처리를 하고 싶다면, 서블릿 필터를 만들어 추가하면 된다.

## 서블릿 컨테이너

이때, 보안을 위해 서블릿필터를 구현한다는 것은 무엇을 의미할까?

서블릿은 서블릿 컨테이너에서 관리되어진다. **즉, 스프링 컨테이너에서 관리되어지지 않기에 스프링 관련 기술을 사용하지 못한다.**

Spring MVC를 공부해보았다면 이해가 쉬울 것이다.  
Spring MVC에서 우리는 비지니스 로직만을 구현하기 위해 서블릿과 컨트롤러를 따로 분리했다. 뿐만 아니라 이러한 방식은 스프링에서 제공하는 다양한 서블릿을 내가 구현하지 않고 쓸 수 있게 만들었다.

그리고 Spring 1.2부터 DelegatingFilterProxy가 나오면서 **서블릿 필터 역시 스프링에서 관리가 가능해졌다.**

## DelegatingFilterProxy 와 FilterChainProxy

> Proxy for a standard Servlet Filter, delegating to a Spring-managed bean that implements the Filter interface.

위는 스프링에서 정의하는 DelegatingFilterProxy이다. DelegatingFilterProxy는 서블릿 컨테이너에서 관리되어지는 서블릿 필터다. 이름에서 알 수 있듯이 해당 필터는 보안을 직접 처리하지 않고, ApplicationContext에 존재하는 springSecurityFilterChain 을 찾아 책임을 위임한다.

이때 FilterChainProxy는 스프링이 제공하는 많은 필터들을 가지고 있고, 내가 사용하고 싶은 필터를 Bean으로 등록하고 커스텀함으로서 사용할 수 도있다.

## 책임 위임

책임을 위임한다는 말이 와닿지 않을 수 있다. 대표적인 서블릿 컨테이너인 톰캣을 예로 들어보자.  
  

SpringContainer를 참조하는 DistpacherServlet을 서블릿 컨테이너에 등록한 뒤, 스프링MVC는 DelegatingFilterProxy를 xml에 추가함으로서 SpringContainer에 있는 빈을 참조할 수 있다.

이때 DelegatingFilterProxy는 Root Application Context를 통해 FilterChainProxy를 참조하는데, root Application Context는 최상단에 있는 context로서 서로 다른 서블릿 컨텍스트에서 공유해야하는 Bean들을 등록해놓고 사용할 수 있기 때문이다.

*다만, 어떻게 root Application Context에 등록된 빈들이 공유 될 수 있는지는 아직 잘 모르겠다..*
 

다음 그림을 보면 이해가 더 쉬울 것이다.
![[스크린샷 2024-09-10 오후 1.29.43.png]]


### 번외. 스프링부트 내장 톰캣에 대해서

**결론 부터 말하면 내장 톰캣은 스프링 컨테이너에 빈으로 등록되어있지 않다.**
나의 의문은 스프링부트에 내장 톰캣이 빈으로 등록되어있다면, DelegatingFilterProxy가 필요하지 않지 않나? 이었다.

세부 코드를 뜯어보고 아니라는 것을 알 수 있었는데, 이는 다음과 같다.

SpringApplication이 실행되면 ServletWebServerFactory를 구현하는 
TomcatServletWebServerFactory에서 서블릿 컨테이너 인스턴스를 생성하고 구성한다. TomcatServletWebServerFactory 의 경우에는 ServletWebServerApplicationContext에 빈으로 등록된다.
다만 ServletWebServerFactory에 의해 생성된 Tomcat은 빈으로 등록되지 않는다.


이후 ApplicationContext의 구현체인 ServletWebServerApplicationContext가 서블릿 컨테이너를 시작한다.
![[스크린샷 2024-09-10 오후 1.30.06.png]]
우의 코드를 보면 createWebserver에서 this.webServer에 getwebServerFactory를 통해 webserver를 가져온다.
TomcatServletWebServerFactory의 getwebserver 코드를 보면 다음과 같다.
![[스크린샷 2024-09-10 오후 1.30.27.png]]


여기서 getTomcatWebServer는 tomcat을 바탕으로 다시 new TomcatWebServer를 반환하는데, 중요한 건 webserver가 빈으로 등록되는게 아니라 별도의 webserver라는 필드에 값이 들어간다는 것이다.  따라서 빈으로 등록된 객체는 스프링의 생명주기와 다른데, 이는 Spring Bean의 생명주기를 공부하며 따로 설명하겠다.

결론은 스프링부트에서 내장 톰캣은 스프링 context 내에 존재하지만, bean으로 등록된 것은 아니기에 delegatingfilterproxy를 통해 내부의 빈을 공유해야하는 건 스프링부트를 사용하지 않는 기본 스프링과 동일하다.

