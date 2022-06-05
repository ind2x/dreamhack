## mongoboard
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

<br>

+ app.15162a1a.js

```javascript
(function(t) {
    function e(e) {
        for (var a, i, l = e[0], s = e[1], u = e[2], d = 0, p = []; d < l.length; d++)
            i = l[d],
            r[i] && p.push(r[i][0]),
            r[i] = 0;
        for (a in s)
            Object.prototype.hasOwnProperty.call(s, a) && (t[a] = s[a]);
        c && c(e);
        while (p.length)
            p.shift()();
        return o.push.apply(o, u || []),
        n()
    }
    function n() {
        for (var t, e = 0; e < o.length; e++) {
            for (var n = o[e], a = !0, l = 1; l < n.length; l++) {
                var s = n[l];
                0 !== r[s] && (a = !1)
            }
            a && (o.splice(e--, 1),
            t = i(i.s = n[0]))
        }
        return t
    }
    var a = {}
      , r = {
        app: 0
    }
      , o = [];
    function i(e) {
        if (a[e])
            return a[e].exports;
        var n = a[e] = {
            i: e,
            l: !1,
            exports: {}
        };
        return t[e].call(n.exports, n, n.exports, i),
        n.l = !0,
        n.exports
    }
    i.m = t,
    i.c = a,
    i.d = function(t, e, n) {
        i.o(t, e) || Object.defineProperty(t, e, {
            enumerable: !0,
            get: n
        })
    }
    ,
    i.r = function(t) {
        "undefined" !== typeof Symbol && Symbol.toStringTag && Object.defineProperty(t, Symbol.toStringTag, {
            value: "Module"
        }),
        Object.defineProperty(t, "__esModule", {
            value: !0
        })
    }
    ,
    i.t = function(t, e) {
        if (1 & e && (t = i(t)),
        8 & e)
            return t;
        if (4 & e && "object" === typeof t && t && t.__esModule)
            return t;
        var n = Object.create(null);
        if (i.r(n),
        Object.defineProperty(n, "default", {
            enumerable: !0,
            value: t
        }),
        2 & e && "string" != typeof t)
            for (var a in t)
                i.d(n, a, function(e) {
                    return t[e]
                }
                .bind(null, a));
        return n
    }
    ,
    i.n = function(t) {
        var e = t && t.__esModule ? function() {
            return t["default"]
        }
        : function() {
            return t
        }
        ;
        return i.d(e, "a", e),
        e
    }
    ,
    i.o = function(t, e) {
        return Object.prototype.hasOwnProperty.call(t, e)
    }
    ,
    i.p = "/";
    var l = window["webpackJsonp"] = window["webpackJsonp"] || []
      , s = l.push.bind(l);
    l.push = e,
    l = l.slice();
    for (var u = 0; u < l.length; u++)
        e(l[u]);
    var c = s;
    o.push([0, "chunk-vendors"]),
    n()
}
)({
    0: function(t, e, n) {
        t.exports = n("56d7")
    },
    "034f": function(t, e, n) {
        "use strict";
        n("64a9")
    },
    3260: function(t, e, n) {
        "use strict";
        n("440b")
    },
    "440b": function(t, e, n) {},
    "56d7": function(t, e, n) {
        "use strict";
        n.r(e);
        var a = n("2b0e")
          , r = n("5f5b")
          , o = function() {
            var t = this
              , e = t.$createElement
              , n = t._self._c || e;
            return n("div", {
                attrs: {
                    id: "app"
                }
            }, [n("Header"), n("router-view")], 1)
        }
          , i = []
          , l = function() {
            var t = this
              , e = t.$createElement
              , n = t._self._c || e;
            return n("div", [n("b-navbar", {
                attrs: {
                    type: "dark",
                    variant: "dark"
                }
            }, [n("b-navbar-brand", {
                attrs: {
                    to: "/"
                }
            }, [t._v("MongoBoard")]), n("b-navbar-toggle", {
                attrs: {
                    target: "nav-collapse"
                }
            })], 1)], 1)
        }
          , s = []
          , u = {
            nama: "Header",
            data() {
                return {}
            }
        }
          , c = u
          , d = n("2877")
          , p = Object(d["a"])(c, l, s, !1, null, null, null)
          , h = p.exports
          , b = {
            name: "App",
            components: {
                Header: h
            }
        }
          , f = b
          , v = (n("034f"),
        Object(d["a"])(f, o, i, !1, null, null, null))
          , m = v.exports
          , _ = n("8c4f")
          , y = function() {
            var t = this
              , e = t.$createElement
              , n = t._self._c || e;
            return n("div", [n("b-table", {
                attrs: {
                    striped: "",
                    hover: "",
                    items: t.items,
                    fields: t.fields
                },
                on: {
                    "row-clicked": t.rowClick
                }
            }), n("b-button", {
                on: {
                    click: t.writeContent
                }
            }, [t._v("Write")])], 1)
        }
          , w = []
          , k = n("bc3a")
          , g = n.n(k);
        const x = g.a.create({
            baseURL: "/api/"
        });
        async function j() {
            return await x.get("board")
        }
        async function C(t) {
            return await x.get("board/" + t)
        }
        async function O(t) {
            return await x.put("board", t)
        }
        var $ = {
            name: "BoardList",
            data() {
                return {
                    fields: [{
                        key: "_id",
                        label: "no",
                        sortable: !0
                    }, {
                        key: "title",
                        label: "Title"
                    }, {
                        key: "author",
                        label: "Author"
                    }, {
                        key: "publish_date",
                        label: "publish_date",
                        sortable: !0
                    }],
                    items: []
                }
            },
            async created() {
                let t = await j();
                this.items = t.data
            },
            methods: {
                rowClick(t) {
                    if (null == t._id)
                        return alert("Secret Document !");
                    this.$router.push({
                        path: "/detail/" + t._id
                    })
                },
                writeContent() {
                    this.$router.push({
                        path: "/write"
                    })
                }
            },
            computed: {
                rows() {
                    return this.items.length
                }
            }
        }
          , B = $
          , S = Object(d["a"])(B, y, w, !1, null, null, null)
          , E = S.exports
          , P = function() {
            var t = this
              , e = t.$createElement
              , n = t._self._c || e;
            return n("div", [n("b-card", [n("div", {
                staticClass: "content-detail-content-info"
            }, [n("div", {
                staticClass: "content-detail-content-info-left"
            }, [n("div", {
                staticClass: "content-detail-content-info-left-subject"
            }, [t._v(t._s(t.title))])]), n("div", {
                staticClass: "content-detail-content-info-right"
            }, [n("div", {
                staticClass: "content-detail-content-info-right-user"
            }, [t._v("Author: " + t._s(t.author))]), n("div", {
                staticClass: "content-detail-content-info-right-created"
            }, [t._v("publish_date: " + t._s(t.publish_date))])])]), n("div", {
                staticClass: "content-detail-content"
            }, [t._v(t._s(t.body))])]), n("b-button", {
                on: {
                    click: t.Back
                }
            }, [t._v("Back")])], 1)
        }
          , M = []
          , T = {
            name: "BoardDetail",
            data() {
                return {
                    id: this.$route.params.id,
                    title: null,
                    author: null,
                    publish_date: null,
                    body: null
                }
            },
            async created() {
                let t = await C(this.$route.params.id);
                this.title = t.data.title,
                this.author = t.data.author,
                this.publish_date = t.data.publish_date,
                this.body = t.data.body
            },
            methods: {
                Back() {
                    this.$router.push({
                        path: "/"
                    })
                }
            }
        }
          , A = T
          , D = (n("3260"),
        Object(d["a"])(A, P, M, !1, null, "72d13cd4", null))
          , H = D.exports
          , L = function() {
            var t = this
              , e = t.$createElement
              , n = t._self._c || e;
            return n("div", [n("b-input", {
                attrs: {
                    placeholder: "Title"
                },
                model: {
                    value: t.title,
                    callback: function(e) {
                        t.title = e
                    },
                    expression: "title"
                }
            }), n("b-input", {
                attrs: {
                    placeholder: "Author"
                },
                model: {
                    value: t.author,
                    callback: function(e) {
                        t.author = e
                    },
                    expression: "author"
                }
            }), n("b-form-textarea", {
                attrs: {
                    placeholder: "Contents",
                    rows: "3",
                    "max-rows": "6"
                },
                model: {
                    value: t.body,
                    callback: function(e) {
                        t.body = e
                    },
                    expression: "body"
                }
            }), n("b-form-checkbox", {
                model: {
                    value: t.secret,
                    callback: function(e) {
                        t.secret = e
                    },
                    expression: "secret"
                }
            }, [t._v(" Secret")]), n("br"), n("b-button", {
                on: {
                    click: t.saveContent
                }
            }, [t._v("Save")]), t._v(" \n  "), n("b-button", {
                on: {
                    click: t.cancle
                }
            }, [t._v("Cancle")])], 1)
        }
          , W = []
          , J = {
            name: "BoardWrite",
            data() {
                return {
                    title: "",
                    author: "",
                    body: "",
                    secret: !1
                }
            },
            methods: {
                async saveContent() {
                    await O({
                        title: this.title,
                        body: this.body,
                        author: this.author,
                        secret: this.secret
                    }),
                    this.$router.push({
                        path: "/"
                    })
                },
                cancle() {
                    this.$router.push({
                        path: "/"
                    })
                }
            }
        }
          , F = J
          , N = Object(d["a"])(F, L, W, !1, null, null, null)
          , R = N.exports
          , U = function() {
            var t = this
              , e = t.$createElement;
            t._self._c;
            return t._m(0)
        }
          , q = [function() {
            var t = this
              , e = t.$createElement
              , n = t._self._c || e;
            return n("div", [n("h1", [t._v("Not Found !")])])
        }
        ]
          , z = {}
          , G = Object(d["a"])(z, U, q, !1, null, null, null)
          , I = G.exports;
        a["default"].use(_["a"]);
        var K = new _["a"]({
            mode: "history",
            routes: [{
                path: "/",
                name: "BoardList",
                component: E
            }, {
                path: "/detail/:id",
                name: "BoardDetail",
                component: H
            }, {
                path: "/write",
                name: "BoardWrite",
                component: R
            }, {
                path: "*",
                component: I
            }]
        });
        n("f9e3"),
        n("2dd8");
        a["default"].config.productionTip = !1,
        a["default"].use(r["a"]),
        new a["default"]({
            router: K,
            render: t=>t(m)
        }).$mount("#app")
    },
    "64a9": function(t, e, n) {}
});
//# sourceMappingURL=app.15162a1a.js.map
```

<br><br>

## Solution
---


