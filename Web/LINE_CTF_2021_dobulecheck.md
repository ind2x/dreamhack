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

<br>

+ www
```javascript
#!/usr/bin/env node

/**
 * Module dependencies.
 */

var app = require('../app');
var debug = require('debug')('doublecheck:server');
var http = require('http');

/**
 * Get port from environment and store in Express.
 */

var port = normalizePort(process.env.PORT || '3000');
app.set('port', port);

/**
 * Create HTTP server.
 */

var server = http.createServer(app);

/**
 * Listen on provided port, on all network interfaces.
 */

server.listen(port);
server.on('error', onError);
server.on('listening', onListening);

/**
 * Normalize a port into a number, string, or false.
 */

function normalizePort(val) {
  var port = parseInt(val, 10);

  if (isNaN(port)) {
    // named pipe
    return val;
  }

  if (port >= 0) {
    // port number
    return port;
  }

  return false;
}

/**
 * Event listener for HTTP server "error" event.
 */

function onError(error) {
  if (error.syscall !== 'listen') {
    throw error;
  }

  var bind = typeof port === 'string'
    ? 'Pipe ' + port
    : 'Port ' + port;

  // handle specific listen errors with friendly messages
  switch (error.code) {
    case 'EACCES':
      console.error(bind + ' requires elevated privileges');
      process.exit(1);
      break;
    case 'EADDRINUSE':
      console.error(bind + ' is already in use');
      process.exit(1);
      break;
    default:
      throw error;
  }
}

/**
 * Event listener for HTTP server "listening" event.
 */

function onListening() {
  var addr = server.address();
  var bind = typeof addr === 'string'
    ? 'pipe ' + addr
    : 'port ' + addr.port;
  debug('Listening on ' + bind);
}

```

<br>

+ 현재 경로는 /```usr/src/app```

<br><br>

## Solution
---



