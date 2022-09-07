## read_flag
---

goahead를 사용하는 웹 서비스 입니다.

주석으로 감싸진 FLAG를 획득하세요.

<br><br>

## Solution
---

왜 어이 없다고 한 지 알 것 같다..

우선 관련 취약점을 찾으려면 깃허브의 issue 부분에서 security alert 부분을 확인해봐야 한다..

하나씩 보다보면 %2e에 관한 취약점이 있다.

<a href="https://github.com/embedthis/goahead/issues/300">https://github.com/embedthis/goahead/issues/300</a>

<br>

요약하면 4와 5 버전에서 ```.jst```를 ```%2ejst```로 했을 때, 겉으로 보기에는 똑같은 의미인데.. ```.jst``` 확장자를 제대로 인식하지 못해서 jst 핸들러를 처리하지 못한다고 한다.

그래서 ```flag%2ejst```로 보내주면 플래그가 보인다..
