## web-HTTP-CLI
---

```python
#!/usr/bin/python3
import urllib.request
import socket

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

def get_host_port(url):
    return url.split('://')[1].split('/')[0].lower().split(':')


with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.bind(('', 8000))
    s.listen()

    while True:
        try:
            cs, ca = s.accept()
            cs.sendall('[Input Example]\n'.encode())
            cs.sendall('> https://dreamhack.io:443/\n'.encode())
        except:
            continue
        while True:
            cs.sendall('> '.encode())
            url = cs.recv(1024).decode().strip()
            print(url)
            if len(url) == 0:
                break
            try:
                (host, port) = get_host_port(url)
                if 'localhost' == host:
                    cs.sendall('cant use localhost\n'.encode())
                    continue
                if 'dreamhack.io' != host:
                    if '.' in host:
                        cs.sendall('cant use .\n'.encode())
                        continue
                cs.sendall('result: '.encode() + urllib.request.urlopen(url).read())
            except:
                cs.sendall('error\n'.encode())
        cs.close()
```

<br><br>

## Solution
---

문제해결 조건은 먼저 ```1) get_host_port(url)의 리턴 값으로 host, port 값이 리턴```되어야 하며, ```2) host 값이 localhost면 안되고```, ```3) host 값이 dreamhack.io가 아니라면 점이 포함되면 안된다.```

살짝의 힌트를 얻어 scheme을 ```file://```로 했을 때, ```urllib.request.urlopen(url).read()```에 file 스키마가 들어가도 요청이 보내진다.

왜냐하면 만약 모듈이 ```requests``` 모듈이었다면 이 모듈은 HTTP만 지원하므로 안됬을 거지만, ```urllib.request```는 url을 여는걸 도와주는 모듈이므로 다양한 scheme을 지원하기 때문에 file 스키마도 가능한 것이다.

<br>

이제 조건을 우회해야 하는데, 우리는 플래그를 가져와야하기 때문에 로컬호스트에 접속해야 한다.

localhost를 우회하는 방식으로 10진수 dot 형식말고 decimal 형태의 ip주소를 이용해서 ```127.0.0.1```을 변환하면 ```2130706433```이 된다.

즉, ```http://127.0.0.1/```은 ```http://2130706433/```과 동일하다.

변환하는 과정은 각각의 숫자를 16진수로 변환하여 전체 16진수값을 정수로 변환하면 된다.

<br>

우리는 리턴 값 2개가 필요하므로 port 번호가 문제인데, 처음엔 ```file://2130706433:80/app/flag.txt```를 하니 에러가 발생했다.

직접 테스트해보니 찾을 수 없다고 뜬다.

port 값에 꼭 어떤 숫자 값이 필요한 것은 아니므로 콜론만 붙여주고 보냈더니 요청이 보내졌다.

따라서 최종 페이로드는 ```file://2130706433:/app/flag.txt```

