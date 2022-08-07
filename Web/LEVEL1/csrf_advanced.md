## CSRF Advanced
<hr style="border-top: 1px solid;"><br>

```python
#!/usr/bin/python3
from flask import Flask, request, render_template, make_response, redirect, url_for
from selenium.webdriver.common.by import By
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

users = {
    'guest': 'guest',
    'admin': FLAG
}

session_storage = {}
token_storage = {}

def read_url(url, cookie={"name": "name", "value": "value"}):
    cookie.update({"domain": "127.0.0.1"})
    options = webdriver.ChromeOptions()
    try:
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
        driver.get("http://127.0.0.1:8000/login")
        driver.add_cookie(cookie)
        driver.find_element(by=By.NAME, value="username").send_keys("admin")
        driver.find_element(by=By.NAME, value="password").send_keys(users["admin"])
        driver.find_element(by=By.NAME, value="submit").click()
        driver.get(url)
    except Exception as e:
        driver.quit()
        # return str(e)
        return False
    driver.quit()
    return True


def check_csrf(param, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
    return read_url(url, cookie)


@app.route("/")
def index():
    session_id = request.cookies.get('sessionid', None)
    try:
        username = session_storage[session_id]
    except KeyError:
        return render_template('index.html', text='please login')

    return render_template('index.html', text=f'Hello {username}, {"flag is " + FLAG if username == "admin" else "you are not an admin"}')


@app.route("/vuln")
def vuln():
    param = request.args.get("param", "").lower()
    xss_filter = ["frame", "script", "on"]
    for _ in xss_filter:
        param = param.replace(_, "*")
    return param


@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param", "")
        if not check_csrf(param):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'


@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    elif request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        try:
            pw = users[username]
        except:
            return '<script>alert("user not found");history.go(-1);</script>'
        if pw == password:
            resp = make_response(redirect(url_for('index')) )
            session_id = os.urandom(8).hex()
            session_storage[session_id] = username
            token_storage[session_id] = md5((username + request.remote_addr).encode()).hexdigest()
            resp.set_cookie('sessionid', session_id)
            return resp 
        return '<script>alert("wrong password");history.go(-1);</script>'


@app.route("/change_password")
def change_password():
    session_id = request.cookies.get('sessionid', None)
    try:
        username = session_storage[session_id]
        csrf_token = token_storage[session_id]
    except KeyError:
        return render_template('index.html', text='please login')
    pw = request.args.get("pw", None)
    if pw == None:
        return render_template('change_password.html', csrf_token=csrf_token)
    else:
        if csrf_token != request.args.get("csrftoken", ""):
            return '<script>alert("wrong csrf token");history.go(-1);</script>'
        users[username] = pw
        return '<script>alert("Done");history.go(-1);</script>'

app.run(host="0.0.0.0", port=8000)
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

비밀번호 변경 페이지를 보면 pw 값과 csrftoken 값을 같이 보내고 있다.

코드를 보면 csrftoken 값이 ```token_storage[session_id] = md5((username + request.remote_addr).encode()).hexdigest()```로 유저명과 이용자 IP 주소를 합친 문자열의 md5 해시 값이다.

따라서 admin으로 로그인 된 사용자의 IP 주소가 ```127.0.0.1```이므로 ```admin127.0.0.1```을 md5 해시하면 ```7505b9c72ab4aa94b1a4ed7b207b67fb```이다.

<br>

payload
: ```<img src="/change_password?pw=123&csrftoken=7505b9c72ab4aa94b1a4ed7b207b67fb">```

<br>

admin 비밀번호가 123으로 변경되었으므로 로그인하면 플래그가 나온다.

<br><br>
<hr style="border: 2px solid;">
<br><br>
