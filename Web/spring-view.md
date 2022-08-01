## spring-view
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

<br>

서버 포트는 8090이다.

<br><br>

## Solution
---





