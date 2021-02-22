---
layout: post
comments: true
title:  "Understanding Cross-Origin Resource Sharing (CORS)"
date:   2021-02-22 20:04:40 +0900
categories: jekyll update
---

안녕하세요, 오늘은 CORS (Cross-Origin Resource Sharing)에 대해서 포스팅해보겠습니다.

<br>
<br>
<hr>

# 판다TF의 계정서비스 REST API 구조

일단, 저희 TF의 프로젝트 구조에 대해서 조금 설명을 드리자면, 계정서비스는 예매서비스와 순전히 JSON을 주고 받는 REST API로 구성되어있습니다.

계정 API는 (현재는) 예매서비스와 같은 호스트 (개발중은 localhost)에 위치해있고, 포트만 다르게 설정해놓았습니다. 예매서비스는 :8080, 계정서비스는 :8081을 사용합니다.

<center>
. . .
</center>

계정 서비스의 API중 하나인 **아이디 중복 확인** 기능을 예를 들어서 설명드리겠습니다.

아이디 중복 확인을 위해 예매서비스가 계정서비스의 `/member/duplicate/{memberId}` 으로 GET 요청을 보냅니다.

서버에서 `memberId` 중복 여부를 체크한 후, 200 OK 메시지와 함께 `{"idAlreadyExists": "true"}`와 같이아이디 중복 여부 Boolean이 JSON으로 response에 돌아옵니다.

즉, `http://localhost:8081/member/duplicate/{memberId}`로 request를 보내면 `true`/`false` 로 돌아오는 response body를 보고 예매서비스에서 이 정보를 이용할 수 있습니다.

<br>
<br>
<hr>

# 계정 API와 통신하려면?

저는 예매서비스 frontend에서 계정API가 제대로 통신이 되지 않아서 버그를 추적하느라 조금 애를 썼습니다.

예매서비스 프론트엔드 자바스크립트로 다음과 같이 계정서비스로 요청을 보냅니다.

``` js
let data = {memberId: "nhnbetherookie"};
let response = await fetch("http://localhost:8081/member/duplicate", {
    method: "POST",
    headers: {
        "Content-Type": "application/json"
    },
    body: JSON.stringify(data)
});
```

이렇게 해서 요청을 보내니, 브라우저에서 다음과 같은 에러가 출력됩니다.
![스크린샷 2021-02-15 오후 3.05.07.png](/assets/img/sprint5/cors_error.png)

CORS를 설정해야한다고 알려주고 있습니다.

<br>
<br>
<hr>

# CORS란?

[CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)는 브라우저에서 서버에 특정요청을 할 때 정해진 origin에서 오는 요청만 받아들일 수 있도록 해주는 보안 메카니즘입니다. (모든 cross-origin 요청이 CORS 설정을 필요로 하지 않지만, 앞서본 `fetch()`는 CORS를 설정해주어야만 합니다.)

CORS를 간단히 설명드리면, 서버는 접근 가능한 origin, http method, http header를 미리 설정해둡니다.

이후에 클라이언트가 서버로 요청을 보낼 때, 서버가 설정해둔 origin, method, header에 하나라도 일치하지 않으면 요청을 reject합니다.

## [여기서 잠깐!]

[origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin)이란 요청의 근원을 뜻하며, (protocol scheme, hostname, port)의 고유한 3-tuple이라고 생각하시면 됩니다. 즉, protocol scheme, hostname, port가 모두 일치해야지 같은 origin이라고 인식됩니다. 저희 케이스에는 port넘버가 다르기 때문에 예매서비스와 계정서비스가 각자 서로 다른 origin이라고 인식됩니다.

<br>
<br>
<hr>

# CORS 절차

CORS는 2단계에 걸쳐서 이루어집니다.

<center>
<img src="https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/preflight_correct.png" width="50%;"> 
<br>
<a>MDN Web Docs</a>에서 인용한 사진
</center>

<br>

## 첫 번째 소통 - Preflight로 허가받기

먼저, **CORS preflight**이라는 요청을 클라언트가 서버로 보냅니다. 이 요청은 클라이언트가 서버한테 보내려는 cross-origin 요청에 대한 사전허가를 받는 요청이라고 생각하시면 됩니다.

이 때, 클라이언트는 OPTION이라는 HTTP Method를 사용하며, `Access-Control-Request-Method`와 `Access-Control-Request-Headers`를 Header에 셋팅해줌으로서 본래의 cross-origin요청에 어떤 HTTP Method와 HTTP Request Header를 사용할 것인지 서버에게 알려줍니다.

서버는 이 preflight요청을 허가해주면, HTTP 2XX (성공)의 Response를 돌려줍니다. 그럼과 동시에 Response Header에 `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Max-Age`을 설정해줌으로서 클라이언트에게 어떤 요청을 accept하는지 알려줍니다.

제가 CORS 설정을 추가해서 예매서비스(8080)에서 `fetch()`로 다시 계정서비스 API(8081)를 호출해보았습니다. 앞서 말씀드린 것처럼 두 단계에 걸쳐서 서버와 소통이 이루어졌는데, 첫번째 preflight request와 response는 다음과 같습니다.

```
Request:
OPTIONS /member/duplicate HTTP/1.1
Host: localhost:8081
...
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type
Origin: http://localhost:8080
...
```
앞서 살펴본바와 같이 HTTP OPTIONS 메소드를 이용해서 소통이 이루어졌고, CORS에 관련된 Header가 교환 된 것을 확인할 수 있습니다.

보시면 클라이언트는 서버에게 POST 요청을 하려고 하고 있고, 또 그 요청에 `content-type` header를 설정할 것이라고 미리 알려주고 있습니다.


```
Response:
HTTP/1.1 200
...
Access-Control-Allow-Origin: http://localhost:8080
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: content-type
Access-Control-Max-Age: 1800
Allow: GET, HEAD, POST, PUT, DELETE, OPTIONS, PATCH
...
```

이에 대해 서버는 [http://localhost:8080](http://localhost:8080)(클라이언트)에서의 POST 요청과 `content-type`을 셋팅하는 것에 대한 허락을 하고 있습니다.

## 두 번째 소통 - 본격적인 req/res

첫번째 request/response로 클라이언트가 허가를 받았다면, 두 번째 request/response에 원래 client가 필요했던 본격적인 소통이 이루어집니다.

제 `fetch()`의 경우, request와 response는 다음과 같습니다:

```
Request:
POST /member/duplicate HTTP/1.1
Host: localhost:8081
...
Origin: http://localhost:8080
Sec-Fetch-Site: same-site
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://localhost:8080/
...
```

```
Response:
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Access-Control-Allow-Origin: http://localhost:8080
Content-Type: application/json
...
```
<br>
이제 클라이언트는 서버로부터 필요한 데이터를 받았으므로 통신이 마무리됩니다.


(참고로, Cross-origin request 중에서도 두 단계를 거치지 않고 preflight를 제외한 한 단계만 거치는 간단한 요청들도 존재합니다. 이런 요청들을 [simple request](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#simple_requests)라고 부릅니다. Simple request는 링크에 명시된 모든 조건을 충족해야만 합니다.)

<br>
<br>
<hr>

# 스프링 부트와 자바스크립트에 CORS 추가하기

## 스프링 부트

스프링 부트에 CORS 설정을 추가해주는 것은 간단합니다. Controller 메소드마다 설정하는 방법과, 전역(global)으로 설정해주는 방법 두가지가 존재합니다.

Controller에 설정하는 방법은 아주 간다합니다. Controller 메소드 위에 `@CrossOrigin(origins = {허가를 원하는 client url})`을 달아주면 끝입니다.

전역으로 설정해주려면, Application 파일의 `main()` 아래에 WebMvcConfigurer을 다음과 같이 Bean으로 등록해주시면 됩니다.

``` java
@Bean
public WebMvcConfigurer corsConfigurer() {
	return new WebMvcConfigurer() {
		@Override
		public void addCorsMappings(CorsRegistry registry) {
			registry.addMapping("/**")
					.allowedOrigins("http://localhost:8080")
					.allowedMethods("GET", "POST", "PUT", "HEAD", "DELETE");
		}
	};
}
```

`allowedOrigins()`, `allowedMethods()`, `allowedHeaders()`, `allowCredentials()`, `maxAge()` 와 같은 메소드를 사용해서 필요에 맞춰서 설정해주시면 됩니다.

자세한 설명은 [이 문서](https://spring.io/guides/gs/rest-service-cors/)를 참고하시면 됩니다.

## 자바스크립트

이제 자바스크립트의 `fetch()`의 옵션에 `mode: "cors"`를 추가하고 호출하면 성공적으로 서버로부터 데이터가 반환됩니다.

<br>
<br>
<hr>

# 마무리

혹시라도 예매서비스의 browser에서 계정서비스로 REST API 호출이 필요하실 경우, 앞서 본 것과 같이 CORS설정을 추가해주셔야 할 것입니다.

설정이 필요하신 분께 이번 포스팅이 도움이 되었기를 바랍니다. 감사합니다!