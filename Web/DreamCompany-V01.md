## DreamCompany-V01
---

세션을 추가하려면 그리고 JWt 토큰을 생성하려면 계정을 만들어야 함.

이 때 세션을 추가하려면 id가 ```guest[0-9]*```로 시작해야 함.

그리고 manage 페이지에 접속하려면 검증을 거쳐서 통과해야 하는데 checkGroup 함수가 true가 되어야 함.

이 함수에서는 id가 ```manager[0-9]*```이어야 함.

따라서 id가 ```guest[0-9]*manager[0-9]*\w```이면 두 정규표현식을 만족하게 됨. -> ```guest1manager1```

이걸로 계정을 생성한 뒤 manage 페이지에 접속할 수 있음.

<br>

```
1	manager	1239asdasd93932WEASDGasdv4234qwegyunbdf4234664
2	fake	qw9eu8dusf8oyudv8yxze7yr62347789a7we9o7ro837qry78y
3	admin	8ee7f20905f333c62c3f137cc0d103f281b37086b5160af5b9b99cacb37dbe0f

#	log_name	log_message
0	admin_add_log	added to admin, time : 1352341 admin password create is 'shs2848divv8ru4uwau3u48sdifjsigjirjgls8bvcawe2' + time
1	user_log	login to user, time : 144354446
2	user_add_log	added to user, time : 144344809
```
