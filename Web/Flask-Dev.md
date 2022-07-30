## Flask-Dev
---

```python
#!/usr/bin/python3
from flask import Flask
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)

@app.route('/')
def index():
	return 'Hello !'

@app.route('/<path:file>')
def file(file):
	return open(file).read()

app.run(host='0.0.0.0', port=8000, threaded=True, debug=True)
```

<br><br>

+ Dockerfile
```
FROM python:3.8

# ENV
ENV user dreamhack
ENV port 8000

# SET packages
#RUN apt-get update -y
#RUN apt-get install -y python-pip python-dev build-essential musl-dev gcc

# SET challenges
RUN adduser --disabled-password $user
ADD ./deploy /app
WORKDIR /app
RUN pip install -r requirements.txt # or # RUN pip install flask
RUN gcc /app/flag.c -o /flag \
    && chmod 111 /flag && rm /app/flag.c

# RUN
USER $user
EXPOSE $port

ENTRYPOINT ["python"]
CMD ["app.py"]
```

<br><br>

## Solution
---




