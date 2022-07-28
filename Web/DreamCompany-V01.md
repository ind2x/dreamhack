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

비밀번호를 구하고 로그인 한 뒤 admin 페이지에 가보면 super_admin 메시지를 확인해야 함을 알 수 있다.

<br>

```javascript
router.get("/admin/get/:report_id", checkToken, (req, res) => {
  const report_id = req.params.report_id;
  res.json({report: adminReport.getReport(report_id)}).status(200);
})
```

<br>

위의 코드를 보면 checkToken을 통과하면 ```/admin/get/6```에 플래그가 있는데 여기를 확인할 수 있다.

checkToken은 토큰이 있고 id가 admin이면 통과되므로 저 페이지를 확인할 수 있는 것이다.

따라서 플래그를 얻을 수 있다.

<br>

그 전에 생각은 토큰만 있으면 admin 페이지에도 접속할 수 있다고 생각했는데 안된 이유가 checkToken의 ```if ((manager === true && !req.path.toLowerCase().includes("admin")) || value === "admin")```이 코드에서 걸린 것이었다.

문제에서 언인텐 코드가 추가됬다는데, 풀이를 보니 그 전에는 ```toLowerCase()```가 없어서 토큰을 얻은 뒤 ```/Admin/get/6```로 접속을 할 수 있던 것이였다.

지금은 admin이 아니면 admin 페이지에 아예 들어갈 수 없으므로 브루트포스로 풀어야 되는 것이다.
