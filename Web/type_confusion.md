## type_confusion
---

```php
<?php
 if (isset($_GET['view-source'])) {
     show_source(__FILE__);
    exit();
 }
 if (isset($_POST['json'])) {
     usleep(500000);
     require("./lib.php"); // include for FLAG.
    $json = json_decode($_POST['json']);
    $key = gen_key();
    if ($json->key == $key) {
        $ret = ["code" => true, "flag" => $FLAG];
    } else {
        $ret = ["code" => false];
    }
    die(json_encode($ret));
 }

 function gen_key(){
     $key = uniqid("welcome to wargame.kr!_", true);
     $key = sha1($key);
     return $key;
 }
?>

<html>
    <head>
        <script src="//ajax.googleapis.com/ajax/libs/jquery/1.8.1/jquery.min.js"></script>
        <script src="./util.js"></script>
    </head>
    <body>
        <form onsubmit="return submit_check(this);">
            <input type="text" name="key" />
            <input type="submit" value="check" />
        </form>
        <a href="./?view-source">view-source</a>
    </body>
</html>
```

<br>

```javascript
var lock = false;
function submit_check(f){
	if (lock) {
		alert("waiting..");
		return false;
	}
	lock = true;
	var key = f.key.value;
	if (key == "") {
		alert("please fill the input box.");
		lock = false;
		return false;
	}

	submit(key);

	return false;
}

function submit(key){
	$.ajax({
		type : "POST",
		async : false,
		url : "./index.php",
		data : {json:JSON.stringify({key: key})},
		dataType : 'json'
	}).done(function(result){
		if (result['code'] == true) {
			document.write("Congratulations! flag is " + result['flag']);
		} else {
			alert("nope...");
		}
		lock = false;
	});
}
```

## Soluiton
---

php의 loose comparison 취약점과 type confusion 취약점이다. 

단순히 ==로 비교를 할 때, php에서는 자동으로 형변환을 한다.

예를 들면, string 12와 integer 12를 비교할 때, 자동으로 string을 integer로 형변환하여 비교를 하게 되어 True가 된다.

이를 이용해서 문제에선 sha1 해쉬 키가 있는데 우리가 입력한 값과 비교를 하여 맞으면 flag를 출력한다.

<br>

하지만 문제에서 입력한 값을 json으로 변환해서 보내주는데, 이러면 값들이 전부 string으로 넘어갈 것이다.

따라서 burp suite로 직접 json 코드를 보내줘야 한다.

<br>

```{"key":0}```을 보내주면 해시 값이 0으로 시작하거나 알파벳으로 시작하면 true가 되어 flag가 출력된다.
