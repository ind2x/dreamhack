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

그래서 bot을 이용해서 접근을 해야한다 생각했다.

<br>

bot을 보면 blog 파라미터에 값을 설정해서 보내는 것을 확인할 수 있는데, 코드를 보면 JSON 값을 parse 한 뒤, ```user['username']```과 ```hostname```을 확인하여 조건을 만족하면 url 값을 보낸다.

이 부분은 다음과 같이 설정할 수 있다. ----> ```javascript://web-noob.kr","username":"hello```

<br>





