---
title : Dreamhack - md5 password (wargame.kr)
date: 2022-05-01 22:50 +0900
categories: [Dreamhack, Webhacking]
tags: [md5 true option vulnerability, wargame.kr md5 password, raw byte sql injection,wargame.kr]
---

## (wargame.kr) md5 password
<hr style="border-top: 1px solid;"><br>

```php
<?php
 if (isset($_GET['view-source'])) {
  show_source(__FILE__);
  exit();
 }

 if(isset($_POST['ps'])){
  sleep(1);
  include("./lib.php"); # include for $FLAG, $DB_username, $DB_password.
  $conn = mysqli_connect("localhost", $DB_username, $DB_password, "md5_password");
  /*
  
  create table admin_password(
   password char(64) unique
  );
  
  */

  $ps = mysqli_real_escape_string($conn, $_POST['ps']);
  $row=@mysqli_fetch_array(mysqli_query($conn, "select * from admin_password where password='".md5($ps,true)."'"));
  if(isset($row[0])){
   echo "hello admin!"."<br />";
   echo "FLAG : ".$FLAG;
  }else{
   echo "wrong..";
  }
 }
?>
<style>
 input[type=text] {width:200px;}
</style>
<br />
<br />
<form method="post" action="./index.php">
password : <input type="text" name="ps" /><input type="submit" value="login" />
</form>
<div><a href='?view-source'>get source</a></div>
```

<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

예전에 풀었던 문제로 취약점은 ```md5($ps, true)``` 부분에서 발생한다.

```md5($ps, true)``` 에서 true 인자가 추가되면 16글자의 raw binary format으로 리턴된다.
: <a href="https://www.php.net/manual/en/function.md5.php" target="_blank">php.net/manual/en/function.md5.php</a>

<br>

raw binary format에 만약 ```'or1```이 포함되어 있다면? sql injection이 될 것이다.
: <a href="https://cvk.posthaven.com/sql-injection-with-raw-md5-hashes" target="_blank">cvk.posthaven.com/sql-injection-with-raw-md5-hashes</a>

<br>

password: ```129581926211651571912466741651878684928```

<br><br>
<hr style="border: 2px solid;">
<br><br>
