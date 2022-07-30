## Flask-Dev
---

```python
#!/usr/bin/python3
from flask import Flask
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)

@app.route('/')
def index():
	return 'Hello !'

@app.route('/<path:file>')
def file(file):
	return open(file).read()

app.run(host='0.0.0.0', port=8000, threaded=True, debug=True)
```

<br><br>

+ Dockerfile
```
FROM python:3.8

# ENV
ENV user dreamhack
ENV port 8000

# SET packages
#RUN apt-get update -y
#RUN apt-get install -y python-pip python-dev build-essential musl-dev gcc

# SET challenges
RUN adduser --disabled-password $user
ADD ./deploy /app
WORKDIR /app
RUN pip install -r requirements.txt # or # RUN pip install flask
RUN gcc /app/flag.c -o /flag \
    && chmod 111 /flag && rm /app/flag.c

# RUN
USER $user
EXPOSE $port

ENTRYPOINT ["python"]
CMD ["app.py"]
```

<br><br>

## Solution
---

코드를 보면 ```debugger=True```가 되어 있다.

이를 못봤지만 문제에서 경로를 뒤에 입력하면 해당 경로에 있는 파일을 읽는데 디버깅 결과를 출력해줘서 알 수 있었다.

그래서 처음엔 php의 file system에서는 wrapper를 사용할 수 있기 때문에 이와 관련된 것이 있는지 살펴보았으나 못찾았다.

그 다음 ```flask debugger true exploit```라고 검색을 하니 RCE 취약점이 나왔다.

<br>

요약하면 ```/console``` 페이지에 접속하면 파이썬 인터프리터를 사용 가능하며, 보안을 위해 PIN 기능을 설정할 수 있는데, 문제에서는 설정이 되어 있고 이 PIN 번호를 알아내야 플래그를 실행 할 수 있는 것이였다.

이 핀 번호는 생성하는 알고리즘이 있는데, 이 알고리즘 코드는 ```/usr/local/lib/python3.8/site-packages/werkzeug/debug/__init__.py```에 있다.


<br><br>

## 취약점 자료
---

<a href="https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/werkzeug" target="_blank">book.hacktricks.xyz/network-services-pentesting/pentesting-web/werkzeug</a>

<a href="https://lactea.kr/entry/python-flask-debugger-pin-find-and-exploit" target="_blank">lactea.kr/entry/python-flask-debugger-pin-find-and-exploit</a>
