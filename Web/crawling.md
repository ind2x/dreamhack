## crawling (내가 못품)
---

```python
#app.py
import socket
import requests
import ipaddress
from urllib.parse import urlparse
from flask import Flask, request, render_template

app = Flask(__name__)
app.flag = '__FLAG__'

def lookup(url):
    try:
        return socket.gethostbyname(url) # url IP 주소 반환
    except:
        return False

def check_global(ip):
    try:
        return (ipaddress.ip_address(ip)).is_global # IP주소가 공인 호스트라면 true 반환
    except:
        return False

def check_get(url):
    ip = lookup(urlparse(url).netloc) #  host:port 값을 인자로 넣어서 IP주소 반환
    if ip == False or ip =='0.0.0.0':
        return "Not a valid URL."
    res=requests.get(url)
    if check_global(ip) == False:
        return "Can you access my admin page~?"
    for i in res.text.split('>'):
        if 'referer' in i:
            ref_host = urlparse(res.headers.get('refer')).netloc
            if ref_host == 'localhost':
                return False
            if ref_host == '127.0.0.1':
                return False 
    res=requests.get(url)
    return res.text

@app.route('/admin')
def admin_page():
    if request.remote_addr != '127.0.0.1':
    		return "This is local page!"
    return app.flag

@app.route('/validation')
def validation():
    url = request.args.get('url', '')
    ip = lookup(urlparse(url).netloc)
    res = check_get(url)
    return render_template('validation.html', url=url, ip=ip, res=res)

@app.route('/')
def index():
    return render_template('index.html')

if __name__=='__main__':
    app.run(host='0.0.0.0', port=3333)
```

<br>

+ Dockerfile

```
FROM python:3.6-alpine

# ENV
ENV user dreamhack
ENV port 3333

# SET packages
RUN apk update

# SET challenges
RUN adduser --disabled-password $user
ADD ./deploy /app
WORKDIR /app
RUN pip install -r requirements.txt && adduser $user
RUN chown $user:$user /app

# RUN
USER $user
EXPOSE $port

ENTRYPOINT ["python"]
CMD ["app.py"]
```

<br><br>

## Solution
---

URI 구조를 보면 ```scheme:[//[user[:password]@]host[:port]][/path][?query][#fragment]```이다.

여기서 두 개의 함수를 통과해야 하는데, ```socket.gethostbyname(url)``` 함수와 ```(ipaddress.ip_address(ip)).is_global``` 함수를 통과해야 한다.

우선 url에 ```user:password@host:port``` 형식으로 입력해보면 예를 들어, ```http://google.com:80@naver.com```을 입력하면 ```naver.com```으로 이동되는 것을 알 수 있다.

이 점을 이용해서 우회를 할 수 있다.

<br>

문제에서 ```http://google.com:80@naver.com```를 입력하면 netloc 값은 ```google.com:80@naver.com```이 된다.

이 값이 lookup 함수의 인자로 들어가는데, 직접 테스트 할 때는 IP주소가 나오지 않아서 혼란스러웠는데.. 문제에서 보면은 출력된 IP 주소는 ```google.com```의 주소인 걸 확인할 수 있다.

아마 ```google.com```까지만 인식되는 것 같다.. (이유는 잘 모르겠다. 직접 할 때는 다 들어가서 안되는 것 같은데 문제에선 왜 되는지..)

<br>

따라서 ```google.com```의 IP를 통해 두 조건을 모두 통과할 수 있게 되고 ```requests.get("http://google.com:80@naver.com")```을 통해 ```naver.com```에 get 요청을 보낸 뒤 ```res.text``` 값을 출력해주는 것이다.

그러므로 코드를 보면 로컬호스트는 3333번 포트에서 열려있으므로 ```http://google.com:80@localhost:3333/admin```을 해주게 되면 결국 요청은 ```http://localhost:3333/admin```으로 가게 되며, 이 요청은 로컬호스트가 요청하는 것이 되므로 admin 페이지에 있는 플래그가 출력되게 되는 것이다..
