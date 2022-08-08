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


