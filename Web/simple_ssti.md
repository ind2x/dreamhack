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

jinja
: <a href="https://jinja.palletsprojects.com/en/3.0.x/" target="_blank">jinja.palletsprojects.com/en/3.0.x/</a>
: <a href="https://tedboy.github.io/jinja2/index.html" target="_blank">tedboy.github.io/jinja2/index.html</a>

<br>

jinja template
: <a href="https://jinja.palletsprojects.com/en/3.0.x/templates/" target="_blank">jinja.palletsprojects.com/en/3.0.x/templates/</a>

<br>

config를 이용하면 된다.
: <a href="https://flask.palletsprojects.com/en/2.1.x/config/" target="_blank">flask.palletsprojects.com/en/2.1.x/config/</a>

<br>

![image](https://user-images.githubusercontent.com/52172169/169646050-b3214099-3ed0-41f6-8955-23442d93b592.png)

<br><br>

Flask의 속성으로 config가 있고, config의 속성으로 secret_key 메소드가 들어 있다.

<br>

jinja variable
<a href="https://jinja.palletsprojects.com/en/3.0.x/templates/#variables" target="_blank">jinja.palletsprojects.com/en/3.0.x/templates/#variables</a>

## Solution 2
---

