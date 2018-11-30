# Module 9: Structured logging

## Apply structured logging to the project

<details>
<summary><b>Create a simple logger</b></summary><p>

1. Add a file `log.js` to the `lib` folder

2. Modify `lib/log.js` to the following

```javascript
const LogLevels = {
  DEBUG : 0,
  INFO  : 1,
  WARN  : 2,
  ERROR : 3
}

// default to debug if not specified
const logLevelName = process.env.log_level || 'DEBUG'

const isEnabled = (level) => level >= LogLevels[logLevelName]

function appendError(params, err) {
  if (!err) {
    return params
  }

  return Object.assign(
    { },
    params || { }, 
    { errorName: err.name, errorMessage: err.message, stackTrace: err.stack }
  )
}

function log (levelName, message, params) {
  if (!isEnabled(LogLevels[levelName])) {
    return
  }

  let logMsg = Object.assign({}, params)
  logMsg.level = levelName
  logMsg.message = message

  console.log(JSON.stringify(logMsg))
}

module.exports = {
  debug: (msg, params) => log('DEBUG', msg, params),
  info: (msg, params) => log('INFO',  msg, params),
  warn: (msg, params, error) => log('WARN',  msg, appendError(params, error)),
  error: (msg, params, error) => log('ERROR', msg, appendError(params, error))
}
```

3. Modify `functions/get-index.js` and replace `console.log` with this new `log` module. Replace the entire `get-index.js` with the following.

```javascript
const fs = require("fs")
const Mustache = require('mustache')
const http = require('superagent-promise')(require('superagent'), Promise)
const Log = require('../lib/log')

const restaurantsApiRoot = process.env.restaurants_api
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']
const ordersApiRoot = process.env.orders_api

let html

function loadHtml () {
  if (!html) {
    Log.info('loading index.html...')
    html = fs.readFileSync('static/index.html', 'utf-8')
    Log.info('loaded')
  }
  
  return html
}

const getRestaurants = async () => {
  const httpReq = http.get(restaurantsApiRoot)
  return (await httpReq).body
}

module.exports.handler = async (event, context) => {
  const template = loadHtml()
  const restaurants = await getRestaurants()
  const dayOfWeek = days[new Date().getDay()]
  const view = { 
    dayOfWeek, 
    restaurants,
    searchUrl: `${restaurantsApiRoot}/search`,
    placeOrderUrl: `${ordersApiRoot}`
  }
  const html = Mustache.render(template, view)
  const response = {
    statusCode: 200,
    headers: {
      'content-type': 'text/html; charset=UTF-8'
    },
    body: html
  }

  return response
}
```

4. Modify `functions/place-order.js` and replace `console.log` with use of the logger.

First, require the `log` module, at the top of the file.

```javascript
const Log = require('../lib/log')
```

Replace the 2 instances of `console.log` with `Log.debug`.

On ln11:

```javascript
Log.debug(`placing order ID [${orderId}] to [${restaurantName}]`)
```

On ln27:

```javascript
Log.debug(`published 'order_placed' event into Kinesis`)
```

5. Modify `functions/notify-restaurant.js` and replace `console.log` with use of the logger.

First, require the `log` module, at the top of the file.

```javascript
const Log = require('../lib/log')
```

Replace the 2 instances of `console.log` with `Log.debug`.

On ln21:

```javascript
Log.debug(`notified restaurant [${order.restaurantName}] of order [${order.orderId}]`)
```

On ln32:

```javascript
Log.debug(`published 'restaurant_notified' event to Kinesis`)
```

6. Run the integration tests

`STAGE=dev REGION=eu-west-1 npm run test`

and see that the functions are now logging in JSON

```
  When we invoke the GET / endpoint
SSM params loaded
invoking via handler function get-index
{"level":"INFO","message":"loading index.html..."}
{"level":"INFO","message":"loaded"}
    ✓ Should return the index page with 8 restaurants (1988ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (226ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
{"level":"DEBUG","message":"notified restaurant [Fangtasia] of order [e0018c8c-e7fe-5930-a298-793dbf3df179]"}
{"level":"DEBUG","message":"published 'restaurant_notified' event to Kinesis"}
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  When we invoke the POST /orders endpoint
invoking via handler function place-order
{"level":"DEBUG","message":"placing order ID [e229e607-e2bf-5a54-b065-ca27be2b7bbb] to [Fangtasia]"}
{"level":"DEBUG","message":"published 'order_placed' event into Kinesis"}
    ✓ Should return 200
    ✓ Should publish a message to Kinesis stream

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
    ✓ Should return an array of 4 restaurants (141ms)


  7 passing (3s)
```

</p></details>

<details>
<summary><b>Disable debug logging in production</b></summary><p>

1. Open `serverless.yml`. Add a `custom` section, this should be at the same level as `provider` and `plugins`.

```yml
custom:
  stage: ${opt:stage, self:provider.stage}
  logLevel:
    prod: INFO
    default: DEBUG
```

`custom.stage` uses the `${xxx, yyy}` to provide fall backs. In this case, we're saying "if a `stage` variable is provided via the CLI, e.g. `sls deploy --stage staging`, then resolve to `staging`; otherwise, fallback to `provider.stage` in this file (hence the `self` reference"

2. Still in the `serverless.yml`, under `provider` section, add the following

```yml
environment:
  log_level: ${self:custom.logLevel.${self:custom.stage}, self:custom.logLevel.default}
```

After this change, the `provider` section should look like this:

```yml
provider:
  name: aws
  runtime: nodejs8.10
  stage: dev
  region: eu-west-1
  environment:
    log_level: ${self:custom.logLevel.${self:custom.stage}, self:custom.logLevel.default}
```

This applies the `log_level` environment variable (used to decide what level the logger should log at) to all the functions in the project (since it's specified under `provider`).

It references the `custom.logLevel` object (with the `self:` syntax), and also references the `custom.stage` value (remember, this can be overriden by CLI options). So when the deployment stage is `prod`, it resolves to `self:custom.logLevel.prod` and `log_level` would be set to `INFO`.

The second argument, `self:custom.logLevel.default` provides the fallback if the first path is not found. If the deployment stage is `dev`, it'll see that `self:custom.logLevel.dev` doesn't exist, and therefore use the fallback `self:custom.logLevel.default` and set `log_level` to `DEBUG` in that case.

This is a nice trick to specify a stage-specific override, but then fall back to some default value otherwise.

</p></details>