# Module 5: Writing acceptance tests

## Add acceptance tests

**Goal:** Write acceptance tests

<details>
<summary><b>Reuse get-index test case for acceptance testing</b></summary><p>

1. Modify `when.js` to add support invoking functions remotely via API Gateway.

Add the following to the top of the file, right after `const _ = require('lodash')`

```javascript
const aws4 = require('aws4')
const URL = require('url')
const http = require('superagent-promise')(require('superagent'), Promise)
const mode = process.env.TEST_MODE
```

Add the following functions

```javascript
const respondFrom = async (httpRes) => {
  const contentType = _.get(httpRes, 'headers.content-type', 'application/json')
  const body = 
    contentType === 'application/json'
      ? httpRes.body
      : httpRes.text

  return { 
    statusCode: httpRes.status,
    body: body,
    headers: httpRes.headers
  }
}

const signHttpRequest = (url, httpReq) => {
  const urlData = URL.parse(url)
  const opts = {
    host: urlData.hostname, 
    path: urlData.pathname
  }

  aws4.sign(opts)

  httpReq
    .set('Host', opts.headers['Host'])
    .set('X-Amz-Date', opts.headers['X-Amz-Date'])
    .set('Authorization', opts.headers['Authorization'])

  if (opts.headers['X-Amz-Security-Token']) {
    httpReq.set('X-Amz-Security-Token', opts.headers['X-Amz-Security-Token'])
  }
}

const viaHttp = async (relPath, method, opts) => {
  const root = process.env.TEST_ROOT
  const url = `${root}/${relPath}`
  console.log(`invoking via HTTP ${method} ${url}`)

  try {
    const httpReq = http(method, url)

    const body = _.get(opts, "body")
    if (body) {      
      httpReq.send(body)
    }

    if (_.get(opts, "iam_auth", false) === true) {
      signHttpRequest(url, httpReq)
    }

    const authHeader = _.get(opts, "auth")
    if (authHeader) {
      httpReq.set('Authorization', authHeader)
    }

    const res = await httpReq
    return respondFrom(res)
  } catch (err) {
    if (err.status) {
      return {
        statusCode: err.status,
        headers: err.response.headers
      }
    } else {
      throw err
    }
  }
}
```

2. **Replace** `when.we_invoke_get_index` with the following, to toggle between invoking function locally and remotely

```javascript
const we_invoke_get_index = async () => {
  const res = 
    mode === 'handler' 
      ? await viaHandler({}, 'get-index')
      : await viaHttp('', 'GET')

  return res
}
```

3. Open `steps/init.js`, and modify the `init` function to add a new `TEST_ROOT` environment variable. Replace the URL with the one that you deployed.

**IMPORTANT**: this URL should be WITHOUT the trailing `/`

```javascript
process.env.TEST_ROOT = "https://xxx.execute-api.eu-west-1.amazonaws.com/dev"
```

4. Open `package.json`, you will see that you already have a `acceptance` script which sets `TEST_MODE` to `http`

```json
"scripts": {
  "sls": "serverless",
  "test": "TEST_MODE=handler ./node_modules/.bin/mocha tests/test_cases --reporter spec",
  "acceptance": "TEST_MODE=http ./node_modules/.bin/mocha tests/test_cases --reporter spec"
}
```

5. Run the acceptance test

`npm run acceptance`

and see that the `get-index` function is logging that it's `invoking via HTTP GET`

```
  When we invoke the GET / endpoint
invoking via HTTP GET https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (2952ms)
```

Great! We have reused the test case for the `get-index` function as an acceptance test that tests the deployed API end-to-end.

</p></details>

<details>
<summary><b>Reuse get-restaurants test case for acceptance testing</b></summary><p>

1. **Replace** `when.we_invoke_get_restaurants` to toggle between invoking function locally and remotely

```javascript
const we_invoke_get_restaurants = async () => {
  const res =
    mode === 'handler' 
      ? await viaHandler({}, 'get-restaurants')
      : await viaHttp('restaurants', 'GET', { iam_auth: true })

  return res
}
```

2. Run the acceptance test

`npm run acceptance`

and see that both `get-index` and `get-restaurants` tests are `invoking via HTTP GET`

```
  When we invoke the GET / endpoint
invoking via HTTP GET https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (1254ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (81ms)
```

</p></details>

<details>
<summary><b>Reuse search-restaurants test case for acceptance testing</b></summary><p>

1. **Replace** `when.we_invoke_search_restaurants` to toggle between invoking function locally and remotely

```javascript
const we_invoke_search_restaurants = async (theme) => {
  const body = JSON.stringify({ theme })

  const res = 
    mode === 'handler'
      ? viaHandler({ body }, 'search-restaurants')
      : viaHttp('restaurants/search', 'POST', { body })

  return res
}
```

2. Run the acceptance tests

`npm run acceptance`

and see that the search-restaurants test case is `invoking via HTTP POST`

```
  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via HTTP POST https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/restaurants/search
    ✓ Should return an array of 4 restaurants (800ms)
```

3. Run the integration tests

`npm run test`

and see that all the tests are still passing

```
  When we invoke the GET / endpoint
invoking via handler function get-index
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (163ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (948ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
notified restaurant [Fangtasia] of order [c465faf8-acf3-5894-b5b1-8b6e6ffeb288]
published 'restaurant_notified' event to Kinesis
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  When we invoke the POST /orders endpoint
invoking via handler function place-order
placing order ID [0c6f739c-1832-5dfe-878f-9519e7d3cbc7] to [Fangtasia]
published 'order_placed' event into Kinesis
    ✓ Should return 200
    ✓ Should publish a message to Kinesis stream

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
    ✓ Should return an array of 4 restaurants (60ms)


  7 passing (1s)
```

</p></details>

<details>
<summary><b>Acceptance test for place-order function</b></summary><p>

When executing the deployed `place-order` function via API Gateway, the function would publish an `order_placed` event to the real Kinesis stream.

To verify that the event is published as expected, you have some options:

* If events are streamed and backed up in S3 (e.g. via Kinesis Firehose, so all events are recorded in a persistent storage), then you can poll S3 for new events. However, this approach can be time-consuming depending on the Firehose configuration, if data are batched in 5 mins intervals then this approach becomes infeasible.

* If events are streamed to another BI platform, such as Google Big Query, in real time, then that is a far better option - to query Big Query for the expected event.

* You can use the AWS SDK to fetch Kinesis records with `kinesis.getRecords`, but this is clumsy as it's a multi-step process that requires you to describe shards and get shard iterator first, and when there are more than 1 shard in the stream it also becomes infeasible to keep polling every shard until you have found the expected event.

For this workshop, we'll take a short-cut and only validate Kinesis was called when executing as an integration test.

1. Open `test_cases/place-order.js`, and **replace** the 2 tests

```javascript
it(`Should return 200`, async () => {
  expect(resp.statusCode).to.equal(200)
})

it(`Should publish a message to Kinesis stream`, async () => {
  expect(isEventPublished).to.be.true
})
```

with the following, so the test case no longer validates Kinesis event is published when running as an acceptance test

```javascript
it(`Should return 200`, async () => {
  expect(resp.statusCode).to.equal(200)
})

if (process.env.TEST_MODE === 'handler') {
  it(`Should publish a message to Kinesis stream`, async () => {
    expect(isEventPublished).to.be.true
  })
}
```

2. Run acceptance test

`npm run acceptance`

and see that you now only have 6 tests (there are 7 for integration tests)

```
  When we invoke the GET / endpoint
invoking via HTTP GET https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/
    ✓ Should return the index page with 8 restaurants (451ms)

  When we invoke the GET /restaurants endpoint
invoking via HTTP GET https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/restaurants
    ✓ Should return an array of 8 restaurants (69ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
notified restaurant [Fangtasia] of order [0498d2a4-5ce6-5c49-9d73-ce818e51c84c]
published 'restaurant_notified' event to Kinesis
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  When we invoke the POST /orders endpoint
invoking via handler function place-order
placing order ID [05632b2a-4aa0-517d-8dc5-dbede4f9d439] to [Fangtasia]
published 'order_placed' event into Kinesis
    ✓ Should return 200

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via HTTP POST https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/restaurants/search
    ✓ Should return an array of 4 restaurants (101ms)


  6 passing (658ms)
```

</p></details>

<details>
<summary><b>Acceptance test for notify-restaurant function</b></summary><p>

We can publish a Kinesis event via the AWS SDK to execute the deployed `notify-restaurant` function. Since this function publishes to both SNS and Kinesis, we have the same conumdrum in verifying that it's producing the expected side-effects as the `place-order` function.

The same options we discussed earlier apply here, with regards to verifying the `restaurant_notified` event is published to Kinesis.

But how do we verify that SNS message has notified? And what if we had used SES as we intended initially?

To verify that an email was received, we could subscribe a test email address to the SNS topic (or whitelist it in the case of SES). Then we can programmatically (e.g. Gmail has an API which we can use to read our emails) check our inbox and see if we had received the notification email.

For this workshop, we'll take a short-cut and skip the test altogether.

1. Modify `test_cases/notify-restaurant.js` so that the whole test case is skipped when running as an acceptance test.

Enclose the whole ```describe(`When we invoke the notify-restaurant function`, () => {``` block in an `if`, like the following

```javascript
if (process.env.TEST_MODE === 'handler') {
  describe(`When we invoke the notify-restaurant function`, () => {
    ...
  })
}
```

2. Run acceptance test

`npm run acceptance`

and see that all the tests are passing, and you're now down to 4 acceptance tests.

</p></details>