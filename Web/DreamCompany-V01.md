account_route

filter를 거친 뒤

계정이 있거나 어드민이면 JWT 생성

ADMIN이면 admin_id 키 값으로 id 설정

---
manage_route

JWT가 null이면 거부, 아니면 검증을 함

검증 시 에러 또는 결과가 null이면 홈으로 리다이렉트
-> none 알고리즘 공격으로 우회 가능?
-> 확인 결과 버전이 8.5.1로 취약점에 안전한 버전이었음.. 뻘짓한거

checkGroup이 트루면 user_id 값을 result.id값으로 설정 후 next()

checkGroup(req,result.id)에서 result.id가 admin이거나
manager에 숫자 영문자 등이 붙고 req.path에 admin이 없으면 됨


----

세션을 추가하려면 그리고 JWt 토큰을 생성하려면 계정을 만들어야 함.

이 때 세션을 추가하려면 id가 ```guest[0-9]*```로 시작해야 함.

그리고 manage 페이지에 접속하려면 검증을 거쳐서 통과해야 하는데 checkGroup 함수가 true가 되어야 함.

이 함수에서는 id가 ```manager[0-9]*```이어야 함.

따라서 id가 ```guest[0-9]*manager[0-9]*\w```이면 두 정규표현식을 만족하게 됨. -> ```guest1manager1```

이걸로 계정을 생성한 뒤 manage 페이지에 접속할 수 있음.

