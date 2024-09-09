> 공식문서 참조  
> [https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html#csrf-token-request-handler-breach](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html#csrf-token-request-handler-breach)

# 혼자 할 때는 잘되는데, 왜 같이 하면 오류가 나지?

프런트엔드, 백엔드로 나누어서 협업을 하다보면 위와 같은 생각이 들 때가 많다.

프로젝트의 복잡성, 소통의 문제 등 이유는 다양할 수 있지만 **로그인에서 문제가 생겼다면 웹, 브라우저, 보안 정책에 대한 지식이 없어서일 가능성이 높다.**

구현에 대한 내용보다는, 왜 그런 문제가 발생했는지, 해결방법으론 무엇이 있을까에 대한 내용을 정리했다.

## 대부분의 문제는 프런트, 백엔드 배포주소가 다르기 때문.

협업을 하다보면 프런트 엔드 서버와 백엔드 서버가 분리되어 있다고 생각하는 이들을 몇 번 봐왔다. 프런트 서버, 백엔드 서버가 따로 있는 것이 아닌, web 서버든, was든 기본적으로 하나의 서버가 클라이언트와 소통해야함을 알야아한다.

내가 백엔드와 프런트엔드의 배포 주소가 다르다면, 어째서 그렇게 고민했는지 답 할 수 있어야한다. 해커톤이나, 간단한 연습 프로젝트의 경우 짧은 시간 내에 구현하는 것이 목표임으로, 이런 경우가 종종 발생하지만, 일반적인 방법이 아니란건 알아두자.

이때 프록시서버나 앞단에 로드밸런서를 둔 것이 아니라면, cors오류는 당연히 발생한다.

> CORS 그거 어노테이션이나 인터셉터에 예외 주소를 추가하면 끝 아님?

**이 방식이 모든 문제를 해결해주지 않는다.** 그 이유는 아래에서 설명하겠지만, 브라우저 보안 정책에 위배되면 얄짤 없다.


## 배포주소가 다르고 + HTTP 일 때

결론 부터 말하자면, **해당 상황에서 JWT 토큰을 Body에 담는 건 절대 불가능하다**.
이를 해결하기 위해선 URL에 담거나, 로그인 과정을 나누는 등 방법을 사용할 수 있다.
해당 방법을 설명하기 전, 의심이 많은 사람들을 위해 불가능한 이유를 천천히 설명해보겠다.
### 예제 상황
1. 배포 주소가 다르다. HTTP이다.
1. Oauth2 로그인을 백엔드에서 마친 후, 다시 프런트의 url로 돌아가려고 한다.
2. 해당 url로 보낼때 jwt token 같이 보내고 싶다.

이때 문제가 되는건 token을 보내는 도메인과, 받는 도메인의 주소가 다르다는 것이다.
프런트에서는 다른 도메인에서 준 인증정보를 신뢰하지 않고, 백엔드에서는 쿠키가 다른 도메인에서 작동할 수 있도록 해야한다.

HTTPs 라면 다음과 같은 해결 방법이 있다.

프론트에서 요청할 때 credentials: include 설정을 하고, 백엔드에서 쿠키에 SameSite 설정을 None으로 변경해야한다. 이때 SameSite=None의 경우에는 쿠키의 Secure 속성이 포함되어야지만 브라우저 보안 정책에 위반되지 않아 제대로 작동 할 수 있다.

다음의 코드는 Cookie의 속성을 설정하는 간단한 방법이다. 예제 코드이기에 실작동은 안하지만, 이런 방식으로 설정도 가능하단 것만 알아두면 충분하다. 

Spring Security의 경우 5.5부터는 `SameSite` 속성을 설정할 수 있는 기능이 추가되었으니 이를 활용해도 좋다.

![[스크린샷 2024-09-03 오전 11.28.34.png]]


**하지만 HTTP 에서는 해당 방법이 안된다.  이유는 `Secure` 속성이 설정된 쿠키는 오직 HTTPS 프로토콜을 통해서만 전송된다. 따라서 http 상황에서는 우리가 redirect 시에 쿠키를 유지하는 것이 불가능하다.**

### SOL: redirect url 주소에 토큰을담는다.

```java
@Tag(name = "User", description = "유저 관련 정적 주소 모음")
@Controller
public class UserController {
	@Operation(summary = "카카오 로그인", description = "카카오 로그인 페이지 완료후 토큰과 함께 프런트로 리다이렉트하기")
	@GetMapping("/oauth/kakao/redirect")
	public RedirectView login(@RequestParam("code") String code,
		HttpServletResponse response) {

		String email = oAuthService.authenticate(code);
		String token = userService.loginOauth2User(email);

		//
		return new RedirectView(frontUrl + "?tokenValue=" + token);
	}
}
```

body가 아닌 url에 담는 방법이다. 브라우저 보안 정책을 우회하는 것이기 때문에 당연히 좋은 방법이 아니다. 특히 HTTP를 사용할 경우 중간에 트래픽을 가로채는 공격자에 의해 쉽게 노출된다. **다만, 구현이 제일 간단하다.** 프런트에서는 해당 url을 받아 파싱만 하면 되고, 백엔드 역시 body가 아닌 url에 담는 작업만 추가하면 된다. 

### SOL2: 프런트에서 Oauth2 로그인 구현
프런트에서 Oauth2 로그인을 진행하는 방법이다. Authorization code 까지만 진행하고 백엔드로 보내는 방법도 있고, 프런트에서 로그인을 끝내는 방법도 존재한다.






db