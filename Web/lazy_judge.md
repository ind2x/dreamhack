## Lazy Judge
<hr style="border-top: 1px solid;"><br>

```python
# flag in /flag
from flask import Flask, render_template, request, session, redirect, url_for
from multiprocessing import Process
import re
import os
import time
import shutil
import tempfile

user_dir = './user_dir/'
app = Flask(__name__)
header = '/*CHECK*/'
solve = 'SolveCode: '
fake_flag = 'YW4gYWwgbHllbyBqdSBqaSBtZWVlZWVyb25nfg==' 
# 디코딩하면 an al lyeo ju ji meeeeerong~

try:
	app.secret_key = os.urandom(24)
	os.mkdir(user_dir)
except:
	pass

no_func_filter =lambda x:re.findall(r'\(*([^)]+)\)',x)
def no_func(st):
	if 'system' in st:
		return False 
	li = no_func_filter(st)
	for t in li:
		t = t.split('(')[0]
		t = t.replace(';','').replace(' ','\n').split('\n')[-1]
		if len(t) > 5:
			return False
	return True

no_name = ['user_dir', 'templates', 'static']
def check_user(uid):
	for t in no_name:
		if t in uid:
			return False
	if uid.isalnum() and 6 < len(uid) < 10:
		return True
	return False

def check_header(st):
	return st[:len(header)] == header

def get_uid():
	if 'uid' in session.keys():
		uid = session['uid']
	else:
		uid = ''
	return uid

def testcase(udir, inp):
	f_testcase = '/tmp/input_%d'%inp
	with open(f_testcase,'w') as f:
		f.write(str(inp)+'\n')
	return os.system('su nobody -s /bin/sh -c "timeout 30s %s < %s >/dev/null"' % (os.path.join(udir,'main'), f_testcase)) >> 8

def submission(uid, udir):
	inp_l = [1,5,10]
	out_l = [1,120,70]
	user_file = user_dir + uid
	for i in range(3):
		if testcase(udir, inp_l[i]) != out_l[i]:
			with open(user_file,'a') as f:
				f.write('%s Wrong Ans\n' % str(time.time()))
			return False

	now = str(time.time()).split('.')[1] + os.urandom(12).encode('hex')
	save_file = os.path.join(udir, now)
	solve_file = os.path.join(udir,'main.c')
	shutil.copy(solve_file,save_file)
	with open(user_file,'a') as f:
		f.write('%s Solve !!! %s <= It is no meaning. Flag is in /flag. "cat /flag"\n' % (time.time(), fake_flag))
		f.write('%s%s\n' % (solve,now))

@app.route('/')
def index():
	uid = get_uid()
	return render_template('index.html', user=uid)

@app.route('/logout')
def logout():
	session.pop('uid', None)
	session.pop('udir', None)
	return redirect(url_for('index')) 

@app.route('/login', methods=['GET', 'POST'])
def login():
	if request.method == 'POST':
		uid = request.form['uid']
		pw = request.form['pw'].encode('hex')
		if not check_user(uid):
			return 'No Hack'

		user_file = user_dir + uid

		if os.path.isfile(user_file) == False:
			return 'Login Fail'

		with open(user_file,'r') as f:
			user_info = f.read().split('\n')[0]

		if pw == user_info.split('|')[0]:
			session['uid'] = uid
			session['udir'] = user_info.split('|')[1]
			return redirect(url_for('index'))
		return 'Login Fail'
	else:
		return render_template('login.html')

@app.route('/reg', methods=['GET', 'POST'])
def reg():
	if request.method == 'POST':
		uid = request.form['uid']
		pw = request.form['pw'].encode('hex')
		if not check_user(uid):
			return 'No Hack'

		user_file = user_dir + uid

		if os.path.isfile(user_file) == False:
			udir = tempfile.mkdtemp()
			os.chmod(udir,0o777) # chmod 777
			main_file = os.path.join(udir,'main.c') # /tmp/tmpabuhsaf/main.c
			with open(main_file,'w') as f:
				f.write(header+'\n#include <stdio.h>\nint main(){\nputs("Hello World");\nreturn 0;\n}')
			with open(user_file,'w') as f:
				f.write(pw + '|' + udir + '\n')
			return '<script>alert("Reg Sucess");location.href="/";</script>'
		else:
			return 'Already Reg'
	else:
		return render_template('reg.html')


@app.route('/prob', methods=['GET', 'POST'])
def prob():
	if get_uid() == '':
		return '<script>alert("No Login");location.href="/";</script>'

	main_file = os.path.join(session['udir'],'main.c')
	with open(main_file,'r') as f:
		main_data = f.read()

	if not check_header(main_data):
		return 'No Hack'

	if request.method == 'POST':
		code = request.form['code']
		if no_func(code):
			if check_header(code):
				with open(main_file,'w') as f:
					f.write(code)
				os.system('su nobody -s /bin/sh -c "cd %s;gcc main.c -o main >/dev/null 2>/dev/null;"' % session['udir'])
				proc = Process(target=submission, args=(session['uid'],session['udir']))
				proc.start()
				time.sleep(1)
				return redirect(url_for('mypage'))
			else:
				return 'Check File Type'
		else:
			return 'No Hack'
	else:
		return render_template('prob.html', main_data=main_data)

@app.route('/mypage')
def mypage():
	if get_uid() == '':
		return '<script>alert("No Login");location.href="/";</script>'

	uid = session['uid']
	udir = session['udir']
	user_file = user_dir + uid
	solve_file = []

	with open(user_file,'r') as f:
		user_log = f.read().split('\n')[1:]

	for t in user_log:
		if t[:len(solve)] == solve:
			solve_file.append(t[len(solve):])

	view = request.args.get('view')
	if view == None:
		return render_template('mypage.html', log=user_log, solves=solve_file)
	else:
		if view in solve_file:
			fname = os.path.join(udir,view)
			with open(fname,'r') as f:
				data = f.read()
			return data
		else:
			return 'No Hack'

app.run(host='0.0.0.0', port=2222, threaded=True)#, debug=True)
```

<br>

+ Dockerfile

```
FROM ubuntu:16.04

ENV WDIR /lazy_judge
ENV RDIR /lazy_judge
COPY ./deploy/lazy_judge $WDIR/

RUN sed -i "s/http:\/\/archive.ubuntu.com/http:\/\/kr.archive.ubuntu.com/g" /etc/apt/sources.list

RUN DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y python2.7 wget gcc gcc-multilib
RUN wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
RUN python2.7 get-pip.py
RUN python2.7 -m pip install flask==1.0.0
COPY ./flag /flag
RUN chmod 400 /flag

RUN chmod 1733 /tmp /var/tmp /dev/shm
RUN chmod 1733 /proc
RUN chmod 600 $WDIR
RUN chmod +x $WDIR/start.sh

CMD ["/lazy_judge/start.sh"]

EXPOSE 2222
```

<br>

```sh
#!/bin/sh
chmod 1733 /proc
cd /lazy_judge
python run.py
sleep infinity
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

```python
no_func_filter =lambda x:re.findall(r'\(*([^)]+)\)',x)
def no_func(st):
	if 'system' in st:
		return False
	li = no_func_filter(st)
	for t in li:
		t = t.split('(')[0]
		t = t.replace(';','').replace(' ','\n').split('\n')[-1]
		if len(t) > 5:
			return False
	return True
```

<br>

```no_func_filter = lambda x:re.findall(r'\(*([^)]+)\)',x)```
: ```)```로 끝나는 문자열을 찾아서 ```)```을 제외한 나머지 문자열 검색
: ```ex) asdf)ewr)123)``` ---> ```['asdf', 'ewr', '123']```

<br>

정리하면 **```no_func``` 함수는 입력된 코드에 system이 있으면 안되며, 함수명의 길이가 5글자를 초과하면 안된다.**

<br>

```prob``` 페이지에 가면 코드를 입력할 수 있다.

페이지를 보면 input은 stdin, output은 return code라고 되어 있다.

<br>

문제 코드를 보면 prob 페이지에서 코드를 제출하면 ```submission``` 함수가 실행이 되는데, 이 함수에서는 ```./udir/main < /tmp/input_{1,5,10}``` 코드를 실행한다.

```/tmp/input_{1,5,10}```에는 ```1,5,10```의 값이 들어가 있는데, 이 값이 stdin이 되는 것이다.

<br>

그리고 ```testcase``` 함수의 리턴 코드를 보면 오른쪽으로 쉬프트 연산을 8번 실행하므로, 이를 고려해서 코드를 짜면 아래와 같다.

<br> 

```c
/*CHECK*/
#include <stdio.h>
int main(){
char a[100];
scanf("%s", a);
int b = atoi(a);
if(b == 1) { return 256>>8; }
else if(b == 5) { return 30720>>8;}
else { return 17920>>8; }
return 0;
}
```

<br>

이렇게 입력하니 ```SolveCode``` 값이 나온다.

<br>

그러면 이제 flag를 어떻게 출력해야 하는가...........

현재 경로는 ```/lazy_judge```이다.

```
flag is in /flag	--> cat /flag 해줘야 함 -> 어떻게 ? 

RUN chmod 400 /flag
RUN chmod 1733 /tmp /var/tmp /dev/shm    --> rwx-wx-wt
RUN chmod 1733 /proc
RUN chmod 600 /lazy_judge  -> rw-------

pwd : /lazy_judge

/lazy_judge - user_dir  --> 루트 권한 , static

/tmp - input_1, input_5, input_10, tempaskdfj (drwx------) - main.c


/login
uid = aaaaaaa
user_file = /lazy_judge/user_dir/aaaaaaa   --> 루트 권한
user_info = pw|udir
session['uid'] = uid
session['udir'] = udir

/reg
uid [user_dir, templates, static] 불가, 숫자와 영어여야 함, 7~9글자

udir = tempfile.mkdtemp()  ---> /tmp/tmpasdf, chmod777   ---> 모두 접근 가능
main_file = /tmp/tmpsafb/main.c      ---> 모두 접근 가능
main_file < #include ~~~~    
user_file <- pw|udir\n   	---> 루트 권한


/prob
main_file = /tmp/tmpsafb/main.c      ---> 모두 접근 가능  -> /tmp/tmpa3Sp6z
main_data = main_file.read()

su nobody -s /bin/sh -c "cd /tmp/tmpasdfj; gcc -o main main.c  
---> main nobody 권한  --> 나머지는 쓰기 권한 없음
submission 함수 실행

testcase 실행
--> su nobody -s /bin/sh -c timeout 30s /tmp/tmpasdfj/main < /tmp/input_1 > /dev/null
--> main 실행파일은 nobody 권한!!!!!!  --> nobody만 rwx 있고 나머지는 쓰기 권한 없음

solve_file = /tmp/tmpsafj/main.c        ----> 모두 접근 가능
save_file = /tmp/tmpsafj/324i0fsafj123   ---> 모두 접근 가능

user_file에 Solve !!!~~ 저장.

/flag 는 권한이 없어서 못읽음 ? 
```

<br>

```
FILE *f = fopen("/flag", "r");   // ---> 권한이 없어서 못읽음
if(f == NULL) { return 0; }
```

<br>

```
FILE *f = fopen("/../input_1", "r");   ----> 안됨
if(f == NULL) { return 0; }

안되는 이유는 실제로 udir 경로에 가서 실행을 하면 되지만,
다른 경로에서 /tmp/tmp123/main 을 실행하면은 f가 NULL이 됨.

따라서 이 문제에서는 절대경로를 입력해줘야 함.
```

<br>

```
execl은 실행이 안되는 듯 함.
execl("/bin/pwd", "/bin/pwd", 0); --> NULL 떠서 Wrong 뜸.
```

<br>

```
/*CHECK*/
#include <stdio.h>
#include <stdlib.h>
int main(){
char a[100], buffer[200]={0};
scanf("%s", a);
int b = atoi(a);
FILE *f = fopen("/flag", "r");
if(f == NULL) { return 0; }
if(b == 1) { return 256>>8; }
else if(b == 5) { return 30720>>8;}
else { return 17920>>8; }
return 0;
}
```

아.. 생각을 못했다.. 

코드를 보면 uid, udir을 세션으로 저장을 하는데, 세션은 쿠키를 기반으로 한다.

클라이언트는 이 쿠키를 가지게 되는데, 이 쿠키는 암호화된 것이 아니라 jwt 토큰처럼 단순히 서명만한 값으로 디코딩하면 값이 보인다.

단, 변조하려면 secret_key ```(app.secret_key)``` 값을 알아야 한다.

<br>

system 함수 구현해서 시도? --> 실패

```c
#include <stdio.h>
#include <unistd.h>

void test(const char* cmd)
{
        pid_t pid;
        pid = fork();
        execl("/bin/sh","sh","-c",cmd,(char*)0);
}
```

<br>

XXE 시도 --> 값에 ```[```를 넣으면 제대로 들어가지 않음;;

```
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="300" version="1.1" height="200">
    <image xlink:href="expect://ls"></image> --> error, ./static/realru.jpeg로 하면 나오는데, 서버 내에서 가져오는건 아니라서..
</svg>
```

