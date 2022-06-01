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

