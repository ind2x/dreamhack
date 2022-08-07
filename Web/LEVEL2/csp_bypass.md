## CSP Bypass
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
    response.headers[
        "Content-Security-Policy"
    ] = f"default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'self' 'nonce-{nonce}'"
    nonce = os.urandom(16).hex()
    return response


@app.route("/")
def index():
    return render_template("index.html", nonce=nonce)


@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    return param


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

```/vuln``` 페이지는 입력 값을 그대로 리턴해주므로 취약점이 발생한다. 

nonce는 예측 불가능하나 같은 출처 내에서 파일을 업로드하거나 우리가 원하는 내용을 반환하는 페이지가 존재한다면 이를 공격에 이용할 수 있다.

<br>

CSP 정책 중 ```script-src 'self' 'nonce-{nonce}```이 있는데 nonce 값은 추측할 수 없으므로 ```script-src 'self'``` 부분을 이용해야 한다.

단순히 ```?param=<script>alert()</script>```를 하면 정책에 위반된다.

하지만 script 태그의 src를 vuln 페이지로 사용함으로써 CSP 정책에서 제한하고 있는 self 출처를 위반하지 않은 채로 우리가 원하는 스크립트를 로드할 수 있다.

<br>

이때 외부 스크립트 파일의 내용에는 ```<script>``` 요소가 포함되어서는 안된다. 또한 src 속성이 있으면 태그 내부의 코드는 무시된다.
: <a href="http://www.tcpschool.com/html-tag-attrs/script-src" target="_blank">tcpschool.com/html-tag-attrs/script-src</a>

<br>

```<script src="/vuln?param=alert()"></script>```를 해주면 alert가 실행된다. 

여기서 src 속성으로 자바스크립트 코드를 불러오는 것이므로 script 태그를 안붙여도 된다.

<br>

payload
: ```<script src="/vuln?param=location.href='/memo?memo='%2bdocument.cookie"></script>```

<br>

+ Flag : ```DH{81e64da19119756d725a33889ec3909c}```

<br><br>
<hr style="border: 2px solid;">
<br><br>
