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
const unhexTable = new Int8Array([
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, // 0 - 15
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, // 16 - 31
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, // 32 - 47
  +0, +1, +2, +3, +4, +5, +6, +7, +8, +9, -1, -1, -1, -1, -1, -1, // 48 - 63
  -1, 10, 11, 12, 13, 14, 15, -1, -1, -1, -1, -1, -1, -1, -1, -1, // 64 - 79
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, // 80 - 95
  -1, 10, 11, 12, 13, 14, 15, -1, -1, -1, -1, -1, -1, -1, -1, -1, // 96 - 111
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, // 112 - 127
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, // 128 ...
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,
  -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1,  // ... 255
]);
```

<br>

```javascript
function unescapeBuffer(s, decodeSpaces) {
  const out = Buffer.allocUnsafe(s.length);
  let index = 0;
  let outIndex = 0;
  let currentChar;
  let nextChar;
  let hexHigh;
  let hexLow;
  const maxLength = s.length - 2;
  // Flag to know if some hex chars have been decoded
  let hasHex = false;
  while (index < s.length) {
    currentChar = StringPrototypeCharCodeAt(s, index);
    if (currentChar === 43 /* '+' */ && decodeSpaces) {
      out[outIndex++] = 32; // ' '
      index++;
      continue;
    }
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
  }
  return hasHex ? out.slice(0, outIndex) : out;
}
```

<br>

해석하면 우선 unhexTable 변수는 0부터 255까지의 인덱스(아스키코드)가 있는데, ```0~9, A~F, a~f```의 인덱스에는 값으로 각각 자신의 10진수 값을 넣어주었고, 나머지에는 -1로 설정해두었다.

그 다음 중간에 퍼센트 처리에 관한 코드를 보면, 가져온 글자가 ```%```이면 그 다음 문자의 아스키 값을 가져와서 hexHigh 또는 hexLow에 설정한다.

hexHigh 또는 hexLow의 값은 무조건 0~15 사이의 숫자가 된다.

예를 들면, ```%ff```를 본다면 ```%```를 확인한 뒤 그 다음 ```f```를 확인하게 되는데, 이 f는 hexHigh 변수에 f의 10진수 값인 15로 설정이 되고, hexHigh 조건문이 성립이 안되므로 그 다음 hexLow 변수에 마지막 f가 담긴다.

그 다음 hexLow 또한 조건문이 성립이 안되므로 ```currentChar = hexHigh * 16 + hexLow``` 코드가 실행되어 ```%ff```의 10진수 값인 255가 된다.

<br>

이제 생각을 해보면.. ```decodeURIComponent```의 디코딩 범위는 **```%00부터 %7F까지```**이다.

따라서 그 다음 ```%80```부터의 값이 들어가게 되면 위의 unescapeBuffer 함수가 실행이 된다.

여기서 가장 중요한 핵심이 되는 부분이 있다.

unescapeBuffer 코드에서 리턴되는 값은 out 변수인데, out의 변수는 **Buffer 클래스**이다.

<a href="https://nodejs.org/api/buffer.html#buffer" target="_blank">Buffer 클래스를 살펴보자</a>

<br>

```
The Buffer class is a subclass of JavaScript's Uint8Array class and extends it with methods that cover additional use cases. 
Node.js APIs accept plain Uint8Arrays wherever Buffers are supported as well.
```

<br>

기본적으로 **Uint8Array 클래스**를 사용한다고 한다. 즉, **UTF-8**이라는 소리다.

따라서 **0~255까지의 값**만 받을 수 있는데, 그 외의 값이 들어오면 **오버플로우**가 발생한다.

아래의 테스트 코드를 확인해보면 알 수 있다.

<br>

```javascript
const b = Buffer.allocUnsafe(2)
b[0]=255
b[1]=256
console.log(b[0], b[1]) // 255, 0
```

<br>

우리는 ```/flag``` 페이지에 들어가야 하므로 ```../```가 필요하다.

```../```는 ```%2e%2e/```이다. 우리는 ```%2e```를 얻어야 한다.

```%2e```는 아스키코드로 46이므로 46을 얻기 위해선 ```x-255 = 46, x=302```이다.

따라서 우리는 유니코드 302 문자인 ```Į```을 이용해서 접근할 수 있게 되었다.

<br>

이제 validate 함수 우회 방법이다.

첫 validate문은 위의 ```Į``` 자체를 넣어줘서 우회할 수 있고, 두 번째 validate문은 위의 parse 메소드 문서를 확인해보면 알겠지만 ```a=1&a=2```가 있다면, ```{a : ['1','2']}```로 변환해준다.

이렇게 되면 두 번째 validate를 우회할 수 있다.

따라서 최종 페이로드는 ```p=good&p=%ff/ĮĮ/ĮĮ/ĮĮ/flag```이다.

<br> 
 
이것도 읽어 보기 <a href="https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf" target="_blank">blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf</a>
