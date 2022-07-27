account_route

filter를 거친 뒤

계정이 있거나 어드민이면 JWT 생성

ADMIN이면 admin_id 키 값으로 id 설정

---
manage_route

JWT가 null이면 거부, 아니면 검증을 함

검증 시 에러 또는 결과가 null이면 홈으로 리다이렉트
-> none 알고리즘 공격으로 우회 가능?

checkGroup이 트루면 user_id 값을 result.id값으로 설정 후 next()

checkGroup(req,result.id)에서 result.id가 admin이거나
manager에 숫자 영문자 등이 붙고 req.path에 admin이 없으면 됨



그리고 /session을 요청할 때, login_session과 login_log 값을 가져옴


----


