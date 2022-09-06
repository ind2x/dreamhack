## PATCH-1
---

주어진 코드를 분석하고, 해당 코드에 존재하는 취약점들을 패치해보세요.

문제에 대한 자세한 설명은 ```/usage``` 페이지를 확인하여 보시기 바랍니다.

모든 패치가 완료되면 플래그를 획득할 수 있습니다.

<br>

```python
#!/usr/bin/python3
from flask import Flask, request, render_template_string, g, session, jsonify
import sqlite3
import os, hashlib

app = Flask(__name__)
app.secret_key = "Th1s_1s_V3ry_secret_key"

def get_db():
  db = getattr(g, '_database', None)
  if db is None:
    db = g._database = sqlite3.connect(os.environ['DATABASE'])
  db.row_factory = sqlite3.Row
  return db

def query_db(query, args=(), one=False):
  cur = get_db().execute(query, args)
  rv = cur.fetchall()
  cur.close()
  return (rv[0] if rv else None) if one else rv

@app.teardown_appcontext
def close_connection(exception):
  db = getattr(g, '_database', None)
  if db is not None:
    db.close()

@app.route('/')
def index():
  return "api-server"

@app.route('/api/me')
def me():
  if session.get('uid'):
    return jsonify(userid=session['uid'])
  return jsonify(userid=None)

@app.route('/api/login', methods=['POST'])
def login():
  userid = request.form.get('userid', '')
  password = request.form.get('password', '')
  if userid and password:
    ret = query_db(f"SELECT * FROM users where userid='{userid}' and password='{hashlib.sha256(password.encode()).hexdigest()}'" , one=True)
    if ret:
      session['uid'] = ret[0]
      return jsonify(result="success", userid=ret[0])
  return jsonify(result="fail")

@app.route('/api/logout')
def logout():
  session.pop('uid', None)
  return jsonify(result="success")

@app.route('/api/join', methods=['POST'])
def join():
  userid = request.form.get('userid', '')
  password = request.form.get('password', '')
  if userid and password:
    conn = get_db()
    cur = conn.cursor()
    cur.execute("Insert into users values(?, ?);", (userid, hashlib.sha256(password.encode()).hexdigest()))
    conn.commit()
    return jsonify(result="success")
  return jsonify(result="error")

@app.route('/api/memo/add', methods=['PUT'])
def memoAdd():
  if not session.get('uid'):
    return jsonify(result="no login")

  userid = session.get('uid')
  title = request.form.get('title')
  contents = request.form.get('contents')

  if title and contents:
    conn = get_db()
    cur = conn.cursor()
    ret = cur.execute("Insert into memo(userid, title, contents) values(?, ?, ?);", (userid, title, contents))
    conn.commit()
    return jsonify(result="success", memoidx=ret.lastrowid)
  return jsonify(result="error")

@app.route('/api/memo/<idx>', methods=['GET'])
def memoView(idx):
  mode = request.args.get('mode', 'json')
  ret = query_db("SELECT * FROM memo where idx=" + idx)[0]
  if ret:
    userid = ret['userid']
    title = ret['title']
    contents = ret['contents']
    if mode == 'html':
      template = ''' Written by {userid}<h3>{title}</h3>
      <pre>{contents}</pre>
      '''.format(title=title, userid=userid, contents=contents)
      return render_template_string(template)
    else:
      return jsonify(result="success",
        userid=userid,
        title=title,
        contents=contents)
  return jsonify(result="error")

@app.route('/api/memo/<int:idx>', methods=['PUT'])
def memoUpdate(idx):
  if not session.get('uid'):
    return jsonify(result="no login")

  ret = query_db('SELECT * FROM memo where idx=?', [idx,])[0]
  userid = session.get('uid')
  title = request.form.get('title')
  contents = request.form.get('contents')

  if ret and title and contents:
    conn = get_db()
    cur = conn.cursor()
    updateRet = cur.execute("UPDATE memo SET title=?, contents=? WHERE idx=?",(title, contents, idx))
    conn.commit()
    if updateRet:
      return jsonify(result="success")
  return jsonify(result="error")
```

<br><br>

## Solution
---

먼저 발견한 취약점은 sql injection 취약점이다.

```ret = query_db(f"SELECT * FROM users where userid='{userid}' and password='{hashlib.sha256(password.encode()).hexdigest()}'" , one=True)```

<br>

보자마자 느낌이 오는 코드로, 밑에 있는 insert 문처럼 바꿔주면 된다.

```ret = query_db("SELECT * FROM users where userid=? and password=?" , (userid,hashlib.sha256(password.encode()).hexdigest()), one=True)```

이렇게 먼저 제출을 해보면 아래와 같이 다른 취약점이 나온다.

<br>

```
[FAIL]

 - SLA(7/7): SLA PASS
 - VULN(5)
   - Hard-coded Key
   - SQL Injection
   - Server-Side Template Injection
   - Cross Site Scripting
   - Memo Update IDOR
```

<br>

취약점이 5개가 있고 내가 패치한 건 sql injection 부문 하나다.

하드코드 키는 app.secret_key를 얘기하는 것이고, SSTI 취약점은 memoView 함수에서 발생한다.

먼저 이 둘을 처리해본다.

<br><br>

### Hard-coded Key
---

hard coded key는 <a href="https://flask.palletsprojects.com/en/2.1.x/config/#SECRET_KEY">flask.palletsprojects.com/en/2.1.x/config/#SECRET_KEY</a>를 보면 된다.

따라서 ```import secrets```를 추가하고 ```app.secret_key = secrets.token_hex()```로 변경해준다.

<br>

```python
import secrets

app.secret_key = secrets.token_hex()
```

<br>

제출 결과 한 개의 취약점만 줄었는데, 맨 처음 패치한 것이 잘못된 듯 하다..

<br>

```
[FAIL]

 - SLA(7/7): SLA PASS
 - VULN(4)
   - SQL Injection
   - Server-Side Template Injection
   - Cross Site Scripting
   - Memo Update IDOR
```

<br><br>

### SQL Injection
---

sql injection 부분이 다른 곳에도 남아있었는데, memoView 부분에서 발생한다.

원래 idx 같은 값들은 앞에 int라고 명시를 해줘야하는데, 명시해주지 않아서 발생하는 것으로 생각된다.

따라서 ```ret = query_db("SELECT * FROM memo where idx=" + idx)[0]```를 ```ret = query_db("SELECT * FROM memo where idx=?", idx)[0]```로 패치해주면 된다.

<br>

```
[FAIL]

 - SLA(7/7): SLA PASS
 - VULN(3)
   - Server-Side Template Injection
   - Cross Site Scripting
   - Memo Update IDOR
```

<br><br>

### SSTI
---

ssti는 아래 코드에서 발생한다.

<br>

```python
if mode == 'html':
      template = ''' Written by {userid}<h3>{title}</h3>
      <pre>{contents}</pre>
      '''.format(title=title, userid=userid, contents=contents)
      return render_template_string(template)
```

<br>

이 코드를 아래와 같이 패치해주면 된다.

<br>

```python
 if mode == 'html':
      template = '''Written by {{userid}}<h3>{{title}}</h3>
      <pre>{{contents}}</pre>
      '''
      return render_template_string(template, title=title, userid=userid, contents=contents)
```

<br>

```
[FAIL]

 - SLA(7/7): SLA PASS
 - VULN(1)
   - Memo Update IDOR
```

<br><br>

### Memo Update IDOR
---

남은 건 이 취약점 밖에 없다.

먼저 IDOR 취약점이 먼저 알아본다.

