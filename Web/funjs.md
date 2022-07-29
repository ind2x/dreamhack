## funjs
---

```html
<html>
    <head>
    <style>*{margin: 0;}</style>
    <script>
    var box;
    window.onload = init;
    function init() {
      box = document.getElementById("formbox");
    }

    function text2img(text){
        var imglist = box.getElementsByTagName('img');
        while(imglist.length > 0) {imglist[0].remove();}
        var canvas = document.createElement("canvas");
        canvas.width = 620;
        canvas.height = 80;
        var ctx = canvas.getContext('2d');
        ctx.font = "30px Arial";
        var text = text;
        ctx.fillText(text,10,50);
        var img = document.createElement("img");
        img.src = canvas.toDataURL();
        box.append(img);
    };

    function main(){
        var _0x1046=['2XStRDS','1388249ruyIdZ','length','23461saqTxt','9966Ahatiq','1824773xMtSgK','1918853csBQfH','175TzWLTY','flag','getElementById','94hQzdTH','NOP\x20!','11sVVyAj','37594TRDRWW','charCodeAt','296569AQCpHt','fromCharCode','1aqTvAU'];
        var _0x376c = function(_0xed94a5, _0xba8f0f) {
            _0xed94a5 = _0xed94a5 - 0x175;
            var _0x1046bc = _0x1046[_0xed94a5];
            return _0x1046bc;
        };
        var _0x374fd6 = _0x376c;
        (function(_0x24638d, _0x413a92) {
            var _0x138062 = _0x376c;
            while (!![]) {
                try {
                    var _0x41a76b = -parseInt(_0x138062(0x17f)) + parseInt(_0x138062(0x180)) * -parseInt(_0x138062(0x179)) + -parseInt(_0x138062(0x181)) * -parseInt(_0x138062(0x17e)) + -parseInt(_0x138062(0x17b)) + -parseInt(_0x138062(0x177)) * -parseInt(_0x138062(0x17a)) + -parseInt(_0x138062(0x17d)) * -parseInt(_0x138062(0x186)) + -parseInt(_0x138062(0x175)) * -parseInt(_0x138062(0x184));
                    if (_0x41a76b === _0x413a92) break;
                    else _0x24638d['push'](_0x24638d['shift']());
                } catch (_0x114389) {
                    _0x24638d['push'](_0x24638d['shift']());
                }
            }
        }(_0x1046, 0xf3764));
        var flag = document[_0x374fd6(0x183)](_0x374fd6(0x182))['value'],
            _0x4949 = [0x20, 0x5e, 0x7b, 0xd2, 0x59, 0xb1, 0x34, 0x72, 0x1b, 0x69, 0x61, 0x3c, 0x11, 0x35, 0x65, 0x80, 0x9, 0x9d, 0x9, 0x3d, 0x22, 0x7b, 0x1, 0x9d, 0x59, 0xaa, 0x2, 0x6a, 0x53, 0xa7, 0xb, 0xcd, 0x25, 0xdf, 0x1, 0x9c],
            _0x42931 = [0x24, 0x16, 0x1, 0xb1, 0xd, 0x4d, 0x1, 0x13, 0x1c, 0x32, 0x1, 0xc, 0x20, 0x2, 0x1, 0xe1, 0x2d, 0x6c, 0x6, 0x59, 0x11, 0x17, 0x35, 0xfe, 0xa, 0x7a, 0x32, 0xe, 0x13, 0x6f, 0x5, 0xae, 0xc, 0x7a, 0x61, 0xe1],
            operator = [(_0x3a6862, _0x4b2b8f) => {
                return _0x3a6862 + _0x4b2b8f;
            }, (_0xa50264, _0x1fa25c) => {
                return _0xa50264 - _0x1fa25c;
            }, (_0x3d7732, _0x48e1e0) => {
                return _0x3d7732 * _0x48e1e0;
            }, (_0x32aa3b, _0x53e3ec) => {
                return _0x32aa3b ^ _0x53e3ec;
            }],
            getchar = String[_0x374fd6(0x178)];
        if (flag[_0x374fd6(0x17c)] != 0x24) {
            text2img(_0x374fd6(0x185));
            return;
        }
        for (var i = 0x0; i < flag[_0x374fd6(0x17c)]; i++) {
            if (flag[_0x374fd6(0x176)](i) == operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i])) {} else {
                text2img(_0x374fd6(0x185));
                return;
            }
        }
        text2img(flag);
    }
    </script>
    </head>
    <body>
        <div id='formbox'>
            <h2>Find FLAG !</h2>
            <input type='flag' id='flag' value=''>
            <input type='button' value='submit' onclick='main()'>
        </div>
    </body>
</html>
```

<br><br>

## Solution
---

기존의 코드에서 moveBox 코드를 제외한 위의 코드를 개발자 도구를 이용해서 분석한다.

```flag``` 변수는 내 입력 값, ```operator```는 각각 합, 차, 곱, xor이 있고, ```getchar```는 ```fromCharcode()```이다.

if문을 개발자도구와 콘솔을 통해 확인해보면 ```flag['length']```가 36이 아니면 NOP!을 반환한다.

입력 값이 36을 넘어가는 시점에서 다시 보면 for문에서 내가 입력한 값에서 한 글자씩 ascii로 변환하는데 이 값이 ```operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i]))```와 같아야 한다.

즉, 우리는 ```operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i]))```의 값을 알면 되는 것이므로 값일 확인해보면 숫자가 나오는데 이 아스키 값을 문자로 변환해주면 플래그가 나온다.

<br>

```javascript
let a= '';
for (var i = 0x0; i < flag[_0x374fd6(0x17c)]; i++) {let b = operator[i % operator[_0x374fd6(0x17c)]](_0x4949[i], _0x42931[i]).toString(); a += String.fromCharCode(b); }
console.log(a);
```

