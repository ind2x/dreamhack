## simple-ssti
---

```python
#!/usr/bin/python3
from flask import Flask, request, render_template, render_template_string, make_response, redirect, url_for
import socket

app = Flask(__name__)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

app.secret_key = FLAG


@app.route('/')
def index():
    return render_template('index.html')

@app.errorhandler(404)
def Error404(e):
    template = '''
    <div class="center">
        <h1>Page Not Found.</h1>
        <h3>%s</h3>
    </div>
''' % (request.path)
    return render_template_string(template), 404

app.run(host='0.0.0.0', port=8000)
```

## Solution 1
---

+ flask, jinja
    + <a href="https://flask.palletsprojects.com/en/2.1.x/" target="_blank">flask.palletsprojects.com/en/2.1.x/</a>
    + <a href="https://jinja.palletsprojects.com/en/3.0.x/" target="_blank">jinja.palletsprojects.com/en/3.0.x/</a>
    + <a href="https://tedboy.github.io/jinja2/index.html" target="_blank">tedboy.github.io/jinja2/index.html</a>

<br>

+jinja template
    + <a href="https://jinja.palletsprojects.com/en/3.0.x/templates/" target="_blank">jinja.palletsprojects.com/en/3.0.x/templates/</a>

<br>

config를 이용하면 된다. (config : <a href="https://flask.palletsprojects.com/en/2.1.x/config/" target="_blank">flask.palletsprojects.com/en/2.1.x/config/</a>)

<br>

Flask 클래스의 속성으로 config 클래스 변수가 있고 (dictionary 이다.), config의 키 값 중 ```secret_key```가 들어 있다.

config는 dictionary 처럼 동작하지만, 추가적인 메소드도 지원한다.
: <a href="https://flask.palletsprojects.com/en/2.1.x/api/#flask.Config" target="_blank">flask.palletsprojects.com/en/2.1.x/api/#flask.Config</a>
: ```ex) from_object(), from_file()```
: ```ex) app.config.from_object('os')```

<br>

```{{ config }}```

![image](https://user-images.githubusercontent.com/52172169/169646050-b3214099-3ed0-41f6-8955-23442d93b592.png)

<br>

+ jinja variable
    + <a href="https://jinja.palletsprojects.com/en/3.0.x/templates/#variables" target="_blank">jinja.palletsprojects.com/en/3.0.x/templates/#variables</a>

<br>

```config.__getitem__('SECRET_KEY')``` 또는 ```config['SECRET_KEY']```

<br><br>

## Solution 2
---

flag.txt 파일 내용을 보는 방법이 있을 것이다.

<br>

+ jinja2, flask API
    + <a href="https://flask.palletsprojects.com/en/2.1.x/api/" target="_blank">flask.palletsprojects.com/en/2.1.x/api/</a>
    + <a href="https://jinja.palletsprojects.com/en/3.0.x/api/" target="_blank">jinja.palletsprojects.com/en/3.0.x/api/</a>

<br>


