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




