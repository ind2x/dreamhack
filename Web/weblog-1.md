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

여기서 성공한 로그(500 에러가 난 로그)를 살펴보면 admin의 비밀번호는 Th1s_1s_Admin_P@SS






