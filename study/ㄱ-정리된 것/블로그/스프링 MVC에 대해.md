
Spring Web MVC는 Spring의 모듈 중 하나이며, 이름에서 볼 수 있듯이 MVC 디자인 패턴과 관련이 있다.

Spring Web MVC가 하는 일을 간단하게 요약하자면 MVC 디자인 패턴을 따르는 웹 애플리케이션을 구현할 수 있도록 돕는 것이라 할 수 있다.

그렇다면 Spring Web MVC는 어떻게 이를 가능하게 하는지 알아보자.

## MVC 디자인 패턴에 대해.
먼저 MVC 디자인 패턴에 대해 말해보자.

MVC는 Model, View , Controller를 의미한다.
다시말해서, 어플리케이션을 Model,View,Controller의 상호연결을 통해 구현하는 디자인 패턴이다.

따라서 Model,View,Controller의 정의를 알고, 이를 구현할 줄 안다면 MVC 패턴에 대해 안다고 할 수 있을 것이다.

- **Model** : *smalltalk*(MVC 패턴을 최초로 사용하고, 자바에 많은 영향을 미친 객체지향언어)에서 정의한 모델의 의미는 사용자 인터페이스와는 독립적이고, 애플리케이션의 데이터, 로직 및 규칙을 직접 관리하는 요소이다.
- **View** : 뷰는 메뉴나 버튼과 같은 완전한 사용자 인터페이스 요소를 나타내며 사용자로부터 입력을 받고 출력을 보여주는 부분을 의미한다.(https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)
- **Controller**: 입력을 수락하고 이를 모델 또는 뷰에 대한 명령으로 변환한다.


이를 웹 환경에서 다시 생각해보자. 우리는 웹 어플리케이션을 만들때 정보를 저장할 데이터베이스가 필요하고, 사용자에게 화면을 보여주기 위해서 html과 css가 필요하며, 사용자의 요청을 처리하기 위한 코드(java,js,python)가 필요하다.

이를 MVC 패턴에 대입해보면 상당히 잘 맞아 떨어지는데,이를 생각해보면 웹 개발을 위해 MVC 디자인 패턴을 사용하는 것은 꽤나 괜찮은 선택으로 보인다.

## 그렇다면 스프링은 MVC 아키텍쳐를 어떻게 구현했는가?

스프링 없이 순수 자바로 MVC 패턴을 구현해보면, 난감한 점이 많다.
Model,View,Controller 객체를 만들다보면 서로 너무 강하게 결합되고 재사용성이 떨어진다는 느낌을 강하게 받는다. 이 과정에서 자연스럽게 frontcontroller 패턴을 적용할 필요성을 느끼게 되는데, 스프링MVC를 사용할 때는 이러한 고민거리가 없다.

그 이유는 내부적으로 이미 frontcontroller에 해당하는 코드가 구현되어있기 때문인데, 그 부분이 바로 DispatcherServlet이다.

## 스프링 Web MVC 의 전체적인 구조


![[스크린샷 2024-09-24 오후 3.35.14.png]]

https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/mvc.html 에서 제공하는 그림이다.

요청이 들어오면 FrontController(DispatcherServlet)에서 해당 요청을 처리할 컨트롤러를 찾는다.
이때 컨트롤러마다 반환하는 타입이 다르기 때문에 이를 처리해줄 핸들러 어댑터를 찾는 과정 또한 존재하는데,
이를 통해 우리가 얻을 수 있는 장점은 다음과 같다.

1. 요청을 받고 처리하는 하나의 서블렛(FrontController)만 사용하면 된다.
2. 중복되는 코드를 많이 줄일 수 있다. 하나의 컨트롤러를 만들때마다 공통된 작업은 FrontController에서 해결할 수 있다.
3. 반환하는 Model을 응답에 맞게 변환하는 것 FrontController에서 처리하는 만큼 유연성이 높아진다.
4. 역할과 책임의 분리가 더 명확하게 되어있다.


## 번외. DispatcherServlet과 톰켓에 대해
스프링부트만 사용해서 내장톰켓에 익숙하다면 톰켓과 DispatcherSevlet의 관련성을 놓치기 쉽다.
톰켓은 웹서버이면서 WAS이다. 따라서 톰켓은 우리가 만든 컨트롤러를 알아야지만 적절한 응답을 줄 수 있다.

하지만 DispathcerSevlet은 톰켓에서 생성된다. 이를 위해 스프링의 DispatcherSevlet을 톰켓에 등록해주는 작업이 필요할 것이다.

톰켓이 순수한 자바코드로 대체되기 전으로 예를 들자면 web.xml에*org.springframework.web.servlet.DispatcherServlet*을 서블렛으로 등록해주는 것과 같은 작업이 이에 해당한다.

추가로, 톰켓이 생성한 DispatcherServlet에는  스프링 컨테이너에서 관리하고 있는 컨트롤러에 접근하기 위해서 스프링이 관리하는 컨텍스트 영역에 접근할 수 있어야한다.

이는 DispatcherServlet이 생성될때 초기화 작업에서 찾을 수 있다.
DispatcherServlet이 상속하고 있는 FramworkServlet을 먼저 확인해보자.
![[스크린샷 2024-09-23 오후 2.26.15.png]]

아래의 코드는 FramworkServlet의 일부이다.


![[스크린샷 2024-09-23 오후 2.27.05.png]]
if (cwac.getParent() == null)에서 확인할 수 있듯이, rootApplicationContext를 서블릿 컨텍스트의 부모로 설정함으로서 RootApplicationContext(애플리케이션 전체에서 공유되는 전역적인 컨텍스트)에 내용에 접근할 수 있다.

이런 작업은 SprintSecurity의 Filter에서도 동일한 양상을 보이는데, 알아두면 좋을 것 같아 번외글을 남긴다.