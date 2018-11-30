# Module 10: Sample debug logs in production

## Sample 1% of debug logs in production

<details>
<summary><b>Install middy, the middleware engine</b></summary><p>

1. Install `middy` as a dependency

`npm install --save middy`

</p></details>

<details>
<summary><b>Create sample logging middleware</b></summary><p>

1. Modify `lib/log.js` to add a new `enableDebug` function

```javascript
function enableDebug() {
  const oldLevel = process.env.log_level
  process.env.log_level = 'DEBUG'

  return () => {
    process.env.log_level = oldLevel
  }
}
```

And add it to the functions exported by the module, in `module.exports` at the bottom of the file

```javascript
module.exports = {
  debug: (msg, params) => log('DEBUG', msg, params),
  info: (msg, params) => log('INFO',  msg, params),
  warn: (msg, params, error) => log('WARN',  msg, appendError(params, error)),
  error: (msg, params, error) => log('ERROR', msg, appendError(params, error)),
  enableDebug
}
```

2. Modify `lib/log.js` so that `logLevelName` is a function and can be changed at runtime

```javascript
const logLevelName = () => process.env.log_level || 'DEBUG'

const isEnabled = (level) => level >= LogLevels[logLevelName()]
```

This allows us to change the log level for a particular invocation, because every time we log the logger would check against the environment variable.

3. Add a `middleware` folder to the project root

4. Add a file `sample-logging.js` to the `middleware` folder

5. Modify `middleware/sample-logging.js` to the following

```javascript
const Log = require('../lib/log')

// config should be { sampleRate: double } where sampleRate is between 0.0-1.0
module.exports = (config) => {
  const sampleRate = config ? config.sampleRate || 0.01 : 0.01 // defaults to 1%
  let rollback = undefined

  const isDebugEnabled = () => {
    return sampleRate && Math.random() <= sampleRate
  }

  return {
    before: (handler, next) => {
      if (isDebugEnabled()) {
        rollback = Log.enableDebug()
      }

      next()
    },
    after: (handler, next) => {
      if (rollback) {
        rollback()
      }

      next()
    },
    onError: (handler, next) => {
      let awsRequestId = handler.context.awsRequestId
      let invocationEvent = JSON.stringify(handler.event)
      Log.error('invocation failed', { awsRequestId, invocationEvent }, handler.error)
      
      next(handler.error)
    }
  }
}
```

6. To make it easy to apply the same 'pattern' to all of our functions, let's create a wrapp factory function. Add a file `wrapper.js` to the `lib` folder.

7. Modify `lib/wrapper.js` to the following

```javascript
const middy = require('middy')
const sampleLogging = require('../middleware/sample-logging')

module.exports = (f) => {
  return middy(f).use(sampleLogging({ sampleRate: 0.1 }))
}
```

</p></details>

<details>
<summary><b>Wrap function handlers</b></summary><p>

1. Modify `functions/get-index.js` to require the `wrapper` module (at the top of the file)

```javascript
const wrap = require('../lib/wrapper')
```

And use it to wrap our handler function. Change `module.exports.handler = async (event, context) => {` to the following (don't forget the closing `)` at the end!)

```javascript
module.exports.handler = wrap(async (event, context) => {
  ...
})
```

2. Repeat step 1 for all the function handlers.

3. Run integration test

`STAGE=dev REGION=eu-west-1 npm run test`

and see that tests are failing...

```
1) When we invoke the GET / endpoint
    Should return the index page with 8 restaurants:
  TypeError: Cannot read property 'statusCode' of undefined
  at Context.it (tests/test_cases/get-index.js:12:16)
  at <anonymous>
  at process._tickDomainCallback (internal/process/next_tick.js:228:7)
```

And there are also unhandle promise errors from `middy`

```
TypeError","errorMessage":"callback is not a function"
```

This is because, `middy` turns our functions into callback style functions so that it's backward compatible with Node 6.10 as well.

So we need to update `steps/when.js` to match this.

4. Modify `steps/when.js` to use `util.promisify` to turn the handler function back to async function

First, require the `util` module at the top.

```javascript
const util = require('util')
```

Then replace the `viaHandler` function with the following

```javascript
const viaHandler = async (event, functionName) => {
  const handler = util.promisify(require(`${APP_ROOT}/functions/${functionName}`).handler)
  console.log(`invoking via handler function ${functionName}`)

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (_.get(response, 'body') && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}
```

5. Rerun integration tests

`STAGE=dev REGION=eu-west-1 npm run test`

and see that all the tests are now passing

6. Deploy the project

`npm run sls -- deploy -s dev -r eu-west-1`

</p></details>