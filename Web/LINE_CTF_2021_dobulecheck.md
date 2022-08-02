## LINE CTF 2021 doublecheck
---

+ app.js
```javascript
const createError = require('http-errors')
const express = require('express')
const bodyParser = require('body-parser')
const path = require('path')
const logger = require('morgan')
const querystring = require('querystring')
const http = require('http')
const process = require('process')

const app = express()
const port = process.env.PORT || '3000'

logger.token('body', (req, res) => req.body.length ? req.body : '-')
app.use(logger(':method :url :status :response-time ms - :res[content-length] :body'))
app.use(bodyParser.text({ type: 'text/plain' }))
app.use(express.static(path.join(__dirname, 'public')))

app.post('/', function (req, res, next) {
  const body = req.body
  if (typeof body !== 'string') return next(createError(400))

  if (validate(body)) return next(createError(403))
  const { p } = querystring.parse(body)
  if (validate(p)) return next(createError(403))

  try {
    http.get(`http://localhost:${port}/api/vote/` + encodeURI(p), r => {
      let chunks = ''
      r.on('data', (chunk) => {
        chunks += chunk
      })
      r.on('end', () => {
        res.send(chunks.toString())
      })
    })
  } catch (error) {
    next(createError(404))
  }
})

const vote = { good: 0, bad: 0 }
app.get('/votes', function (req, res, next) {
  res.json(vote)
})

// internal apis
app.get('/api/vote/:type', internalHandler, function (req, res, next) {
  if (req.params.type === 'bad') vote.bad += 1
  else vote.good += 1
  res.send('ok')
})

app.get('/flag', internalHandler, function (req, res, next) {
  const flag = process.env.FLAG || 'DH{****}'
  res.send(flag)
})

// catch 404 and forward to error handler
app.use(function (req, res, next) {
  next(createError(404))
})

// error handler
app.use(function (err, req, res, next) {
  res.status(err.status || 500)
  res.send(err.message)
})

function internalHandler (req, res, next) {
  if (req.ip === '::ffff:127.0.0.1') next()
  else next(createError(403))
}

function validate (str) {
  return str.indexOf('.') > -1 || str.indexOf('%2e') > -1 || str.indexOf('%2E') > -1
}

module.exports = app
```

<br><br>

## Solution
---

플래그를 얻기 위해선 ```/flag```로 요청을 보내야 하는데, p 값에 점과 ```%2e, %2E```가 있으면 validate 함수에 걸려서 403 에러를 발생시킨다.

즉, validate 함수를 우회하는게 핵심이다.

그럼 어떻게 우회를 하는가..

<br>

ㅎㅎ.. 결국 내가 풀지는 못했지만 취약점이라 할 수는 없겠지만, 공식문서를 보고 코드를 분석하고 이해하여 풀 수도 있다는 과정을 배웠다고 생각한다.

<br>

```querystring.parse``` 문서 : <a href="https://nodejs.org/api/querystring.html" target="_blank">nodejs.org/api/querystring.html</a>

문서를 보면 parse 설명의 끝 부분을 보면 다음과 같이 써있다.

<br>

```
By default, percent-encoded characters within the query string will be assumed to use UTF-8 encoding. 

If an alternative character encoding is used, then an alternative decodeURIComponent option will need to be specified:
```

<br>

기본적으로 퍼센트인코딩 된 문자는 UTF-8 인코딩을 사용한 것으로 간주하며, 다른 문자 인코딩을 사용한다면 ```decodeURIComponent``` 옵션을 지정해야 한다.

```decodeURIComponent``` 옵션을 보면 기본적으로 ```querystring.unescape()```를 사용한다고 한다.

즉, 퍼센트로 인코딩된 문자를 디코딩할 때, ```querystring.unescape()```를 사용한다고 한다.

```querystring.unescape()```에 대해 보면 아래와 같다.

<br>

```
The querystring.unescape() method performs decoding of URL percent-encoded characters on the given str.

The querystring.unescape() method is used by querystring.parse() and is generally not expected to be used directly. 

It is exported primarily to allow application code to provide a replacement decoding implementation 
if necessary by assigning querystring.unescape to an alternative function.

By default, the querystring.unescape() method will attempt to use the JavaScript built-in decodeURIComponent() method to decode. 

If that fails, a safer equivalent that does not throw on malformed URLs will be used.
```

<br>

정리하면 일반적으로 직접 사용되지 않고, ```querystring.parse```에서 사용되며 **기본적으로 자바스크립트의 내장 함수인 decodeURIComponent()를 디코딩 시 사용**하며, 실패 시 안전한 형식의 URL을 사용(?) 한다고 한다.

```decodeURIComponent()``` 함수는 이스케이프된 문자열을 각각의 자신을 나타내는 문자로 변환해주는 함수이다.

잘못 사용되면  URIError ( " malformed URI sequence ") 예외를 발생시킨다.

<br>

이제 소스코드를 분석해본다.

Link : <a href="https://github.com/nodejs/node/blob/5011009a593437d3e4ab157d448cc464c93c8cc5/lib/querystring.js" target="_blank">github.com/nodejs/node/blob/5011009a593437d3e4ab157d448cc464c93c8cc5/lib/querystring.js</a>

<br>

먼저 unescape 함수를 보면 설명한 대로 ```decodeURIComponent``` 함수를 리턴하고 예외발생 시 ```QueryString.unescapeBuffer(s, decodeSpaces).toString()```를 불러온다.

<br>

```javascript
function qsUnescape(s, decodeSpaces) {
  try {
    return decodeURIComponent(s);
  } catch {
    return QueryString.unescapeBuffer(s, decodeSpaces).toString();
  }
}
```

<br>

이제 <a href="https://github.com/nodejs/node/blob/5011009a593437d3e4ab157d448cc464c93c8cc5/lib/querystring.js#L271" target="_blank">parse 코드</a>를 보자.

<br>

```javascript
let decode = QueryString.unescape;
if (options && typeof options.decodeURIComponent === 'function') {
    decode = options.decodeURIComponent;
}
const customDecode = (decode !== qsUnescape);
```

<br>

설명한 대로 기본은 decodeURIComponent이며 다른걸 사용하면 옵션을 설정해야하므로 if문에서 그걸 확인한다.

<br>

어느 정도 파악을 했으니, 이제 unescape 했을 때 예외가 발생했을 때의 <a href="https://github.com/nodejs/node/blob/5011009a593437d3e4ab157d448cc464c93c8cc5/lib/querystring.js#L80" target="_blank">코드</a> (```QueryString.unescapeBuffer(s, decodeSpaces).toString()```)를 보자.

<br>

```javascript
if (currentChar === 37 /* '%' */ && index < maxLength) {
      currentChar = StringPrototypeCharCodeAt(s, ++index);
      hexHigh = unhexTable[currentChar];
      if (!(hexHigh >= 0)) {
        out[outIndex++] = 37; // '%'
        continue;
      } else {
        nextChar = StringPrototypeCharCodeAt(s, ++index);
        hexLow = unhexTable[nextChar];
        if (!(hexLow >= 0)) {
          out[outIndex++] = 37; // '%'
          index--;
        } else {
          hasHex = true;
          currentChar = hexHigh * 16 + hexLow;
        }
      }
    }
    out[outIndex++] = currentChar;
    index++;
```

<br>

일부분만 가져와서 보면 



<br>

이제 생각을 해보면.. ```decodeURIComponent```의 디코딩 범위는 **%00부터 %7F까지**이다.



