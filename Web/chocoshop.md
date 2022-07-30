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

<br>

```javascript
<script>
Vue.use(Vuex);

function checkStatus(res) {
  if (res.status >= 200 && res.status < 300) {
    return res;
  }

  var error = new Error(res.statusText);
  error.res = res;
  throw error;
}

function parseJSON(res) {
  return res.json();
}


const store = new Vuex.Store({
  state: {
    page: null,
    session: '',
    user: null,
  },

  mutations: {
    setPage(state, page) {
      state.page = page;
    },

    setSessionId(state, sessionId) {
      window.localStorage.setItem('session-id', sessionId);
      state.session = sessionId;
    },

    setUser(state, user) {
      state.user = user;
    },
  },

  actions: {
    init(context) {
      const sessionId = window.localStorage.getItem('session-id');
      context.commit('setSessionId', sessionId);

      if (sessionId) {
        context.dispatch('getMe')
          .then(() => {
            context.commit('setPage', 'has-session');
          })
          .catch(err => {
            context.commit('setSessionId', '');
            context.commit('setPage', 'no-session');
          });
      } else {
        context.commit('setPage', 'no-session');
      }
    },

    navigate(context, page) {
      context.commit('setPage', page);
    },

    getSessionId(context) {
      return fetch('/session')
        .then(checkStatus)
        .then(parseJSON)
        .then(body => {
          context.commit('setSessionId', body.session);
          context.dispatch('getMe');
        });
    },

    getMe(context) {
      return fetch('/me', { headers: { Authorization: context.state.session } })
        .then(checkStatus)
        .then(parseJSON)
        .then(body => {
          context.commit('setUser', body);
        });
    },

    claimCoupon(context) {
      return fetch('/coupon/claim', { headers: { Authorization: context.state.session } })
        .then(checkStatus)
        .then(parseJSON)
        .then(body => {
          return body.coupon;
        });
    },

    submitCoupon(context, coupon) {
      return fetch('/coupon/submit', { headers: { Authorization: context.state.session, coupon: coupon } })
        .then(checkStatus)
        .then(parseJSON)
        .then(body => {
          return body.status;
        });
    },

    claimProduct(context, id) {
      return fetch(`/${id}/claim`, { headers: { Authorization: context.state.session } })
        .then(checkStatus)
        .then(parseJSON)
        .then(body => {
          return body.message;
        });
    },
  }
});


Vue.component('app-header', {
  template: `
    <header class="app-header">
      <nav class="navbar is-dark">
        <div class="navbar-brand">
          <a href="/" class="navbar-item">Chocoshop</a>
        </div>

        <div v-if="user" class="navbar-end">
          <div class="navbar-item">
            {{ user.uuid }}
          </div>
          <div class="navbar-item">
            £{{ user.money }}
          </div>
        </div>
      </nav>
    </header>
  `,

  computed: {
    user() { return this.$store.state.user; },
  },
});

Vue.component('no-session', {
  template: `
    <div>
      <h1 class="title">Session Required</h1>
      <p class="mb-4">Session is required for further access</p>
      <button class="button is-info is-large is-fullwidth" @click="acquireSession">Acquire Session</button>
    </div>
  `,

  methods: {
    acquireSession() {
      this.$store.dispatch('getSessionId');
      this.$store.dispatch('navigate', 'has-session');
    },
  },
});

Vue.component('has-session', {
  template: `
    <div>
      <button class="button is-success is-large" @click="navigate('shop')">SHOP</button>
      <button class="button is-warning is-large" @click="navigate('mypage')">MYPAGE</button>
    </div>
  `,

  methods: {
    navigate(page) {
      this.$store.dispatch('navigate', page);
    },
  }
});

Vue.component('page-shop', {
  template: `
    <div>
      <h2 class="title">Welcome to Chocoshop!</h2>

      <hr>

      <div class="products">
        <div v-for="product in products" :key="product.id" class="product" @click="claimProduct(product.id)">
          <img :src="\`/images/\${product.id}.jpg\`">
          <h3 class="name">{{ product.name }} (£{{ product.price }})</h3>
          <p class="description">{{ product.description }}</p>
        </div>
      </div>
    </div>
  `,

  data() {
    return {
      products: [
        { id: 'pepero', name: 'Pepero', description: 'Do you guys not have Peperos?', price: 1500 },
        { id: 'flag', name: 'Flag', description: 'Get your flag!', price: 2000 },
      ],
    }
  },

  methods: {
    claimProduct(id) {
      this.$store.dispatch('claimProduct', id)
        .then(message => {
          alert(message);
          this.$store.dispatch('getMe');
        })
        .catch(err => {
          if (err.res) {
            err.res.json().then(body => {
              alert(body.message);
            });
          } else {
            alert(err.message);
          }
        });
    },
  },
});

Vue.component('page-mypage', {
  template: `
    <div>
      <h2 class="title">My Page</h2>

      <hr class="my-6">

      <h3 class="subtitle">Coupon Claim</h3>
      <button class="button is-info is-large is-fullwidth" @click="claimCoupon">Claim</button>
      <textarea v-model="couponClaimed" class="textarea" type="text" placeholder="Your Coupon" readonly></textarea>

      <hr class="my-6">

      <h3 class="subtitle">Coupon Submit</h3>
      <textarea v-model="couponSubmit" class="textarea" type="text" placeholder="Enter Your Coupon"></textarea>
      <button class="button is-success is-large is-fullwidth" @click="submitCoupon">Submit</button>
    </div>
  `,

  data() {
    return {
      couponClaimed: '',
      couponSubmit: '',
    };
  },

  methods: {
    claimCoupon() {
      this.$store.dispatch('claimCoupon')
        .then(coupon => {
          this.couponClaimed = coupon;
          this.$store.dispatch('getMe');
        })
        .catch(err => {
          if (err.res) {
            err.res.json().then(body => {
              alert(body.message);
            });
          } else {
            alert(err.message);
          }
        });
    },

    submitCoupon() {
      this.$store.dispatch('submitCoupon', this.couponSubmit)
        .then(coupon => {
          alert('Coupon submitted!');
          this.couponSubmit = '';
          this.$store.dispatch('getMe');
        })
        .catch(err => {
          if (err.res) {
            err.res.json().then(body => {
              alert(body.message);
            });
          } else {
            alert(err.message);
          }
        });
    },
  },
});

Vue.component('main-content', {
  template: `
    <div class="page-content">
      <no-session v-if="page === 'no-session'" />
      <has-session v-if="page === 'has-session'" />
      <page-shop v-if="page === 'shop'" />
      <page-mypage v-if="page === 'mypage'" />
    </div>
  `,

  computed: {
    page() { return this.$store.state.page; },
  },
});

const app = new Vue({
  el: '#app',
  store: store,

  created() {
    this.$store.dispatch('init');
  },
});
</script>
```

<br><br>

## Solution
---



