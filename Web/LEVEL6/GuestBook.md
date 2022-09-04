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

근데 풀이를 보면 Dom Clobbering으로도 풀 수가 있었다..
