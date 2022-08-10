## [wargame.kr] dun worry about the vase
---

Description
Do you know about "padding oracle vulnerability" ?

<br><br>

## Solution
---

개념을 이해하기 위해 두 개의 사이트를 보고 공부

<a href="https://core-research-team.github.io/2020-03-30/Padding-Oracle-Attack" target="_blank">core-research-team.github.io/2020-03-30/Padding-Oracle-Attack</a>

<a href="https://bperhaps.tistory.com/entry/오라클-패딩-공격-기초-설명-Oracle-Padding-Attack" target="_blank">bperhaps.tistory.com/entry/오라클-패딩-공격-기초-설명-Oracle-Padding-Attack</a>  ----> BEST 설명, 이거 보고 풀면 됨


<br>

위의 사이트에서 공부를 하고 나서 그대로 똑같이 따라서 풀면 된다.

그 내용을 바탕으로 코드를 짰다.

<br>

```python
from binascii import hexlify, unhexlify
from base64 import b64decode, b64encode
from requests import get
from urllib.parse import quote

# 6PXtWBmnz0A=XjkWTk/QkMI=

IV = str(hexlify(b64decode('6PXtWBmnz0A=')),'ascii') # e8f5ed5819a7cf40
cipher = 'XjkWTk/QkMI='


url = 'http://host3.dreamhack.games:{port}/main.php'

for i in range(0,256):
    hex_IV = format(i,'02x')
    IV_check = 'e8f5ed5819a7cf' + hex_IV + '' # IV_check 값을 계속 변경해줘야 함
    cookie = b64encode(unhexlify(IV_check)).decode('latin-1') + cipher
    print(hex_IV + " : " + quote(cookie))
    res = get(url, cookies={"L0g1n":quote(cookie)})
    if('alert("invalid user.")' in res.text):
        print("Find Hex!! ---> {}".format(hex_IV))
        print("So, IV_PLAINTEXT is {}".format(hex(int(hex_IV,16)^0x1))) ## xor 값도 1씩 증가
        break
```

<br>

자동으로 뽑아오는 코드는 못짰지만, 코드에서 일부 값을 변경해주면서 하나씩 찾아갔다.

IV_check 변수는 브루트포싱을 할 IV 바이트 부분에 0x00부터 0xff까지의 값을 넣어주는 것이다.

따라서 맨 끝 바이트부터 해야하므로 ```e8f5ed5819a7cf + hex_IV```가 되며, 우리가 찾아야 할 값인 ```IV XOR PLAINTEXT```의 값에서 0x1을 넣어주면 된다.

각각의 값들을 변경해주면서 찾아가면 패딩이 3바이트임을 알게 된다.

그리고 우리는 평문을 안다. (guest)

따라서 guest를 hex로 바꾼 뒤, 각각의 IV 값과 xor을 해주면 ```IV XOR PLAINTEXT``` 값을 구할 수 있다.

내 기준으로 ```0x8f80882b6da4cc43```이 나왔다.

<br>

우리는 admin 세션을 구해야 하므로 ```IV XOR PLAINTEXT```과 IV 값을 XOR 했을 때, admin이 나와야 한다.

따라서 admin일 때의 IV를 구해줘야 한다.

<br>

admin은 ```0x61646d696e```이므로 패딩을 해주면 ```0x61646d696e030303```이 된다.

이 값과 ```IV XOR PLAINTEXT```을 XOR 해주면 admin일 때의 IV 값을 구할 수 있으므로 그 결과 ```0xeee4e54203a7cf40```이 나온다.

이 값을 b64 인코딩해줘서 보내주면 된다.

<br>

```7uTlQgOnz0A=XjkWTk/QkMI=```
