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

아래의 로그는 database 명의 첫 글자를 알아낸 로그 기록이다.

<br>

```
172.17.0.1 - - [02/Jun/2020:09:11:09 +0000] "GET /board.php?sort=if(ord(substr(database(),%201,1))=115,%20(select%201%20union%20select%202),%200) HTTP/1.1" 500 1192 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
```

<br>

