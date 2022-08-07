## jff3_magic
---

hint가 swp라고 한다. 

따라서 ```/.index.php.swp```라고 하면 index.php.swp 파일을 다운로드 하게 된다.

<br>

```php
hash('haval128,5',$_POST['pw'],false) == mysqli_real_escape_string($connect, $userinfo['pw']))
```

<br>

파일이 원래 깨지는건지는 모르겠으나, swp 파일에 위와 같은 코드가 있다.

즉, 내가 입력한 pw 값을 ```haval128,5```로 해시한 후 ```userinfo['pw']```와 비교하여 같아야 한다.

문제 제목에도 써있지만 이 문제는 magic hash를 이용하는 문제이다.

따라서 ```haval128,5```의 magic number인 ```	115528287```을 pw로 입력해주면 플래그가 나온다.
