# Module 4: Writing integration tests

## Add integration tests

**Goal:** Write integration tests

In the demo project, add the basic folder structure for our tests

```
tests
  -- steps
  -- test_cases
```

We'll add the test cases to the `test_cases` folder, and the `when` and `given` steps in the `steps` folder.

When you ran `npm install` in the last module, you have already installed:

* `chai`
* `mocha`
* `cheerio`
* `lodash`

amongst others, so we can use them in our tests.

<details>
<summary><b>Add test case for get-index</b></summary><p>

1. Add `get-index.js` file under `test_cases`

2. Modify `get-index.js` to the following

```javascript
const { expect } = require('chai')
const cheerio = require('cheerio')
const when = require('../steps/when')

describe(`When we invoke the GET / endpoint`, () => {
  it(`Should return the index page with 8 restaurants`, async () => {
    const res = await when.we_invoke_get_index()

    expect(res.statusCode).to.equal(200)
    expect(res.headers['content-type']).to.equal('text/html; charset=UTF-8')
    expect(res.body).to.not.be.null

    const $ = cheerio.load(res.body)
    const restaurants = $('.restaurant', '#restaurantsUl')
    expect(restaurants.length).to.equal(8)
  })
})
```

3. Add `when.js` file under `steps`

4. Modify `when.js` to the following

```javascript
const APP_ROOT = '../../'
const _ = require('lodash')

const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler

  const context = {}
  const response = await handler(event, context)
  const contentType = _.get(response, 'headers.content-type', 'application/json');
  if (response.body && contentType === 'application/json') {
    response.body = JSON.parse(response.body);
  }
  return response
}

const we_invoke_get_index = () => viaHandler({}, 'get-index')

module.exports = {
  we_invoke_get_index
}
```

5. In your `package.json` you already have the following `scripts`

```json
"scripts": {
  "sls": "serverless",
  "test": "TEST_MODE=handler ./node_modules/.bin/mocha tests/test_cases --reporter spec --timeout 5000",
  "acceptance": "TEST_MODE=http ./node_modules/.bin/mocha tests/test_cases --reporter spec --timeout 5000"
},
```

Which lets you run the integration tests with

`npm run test`

and see that the test fails with the error 

```
1) When we invoke the GET / endpoint
      Should return the index page with 8 restaurants:
   TypeError: Parameter "urlObj" must be an object, not undefined
```

The `get-index` function needs a number of environment variables.

6. Add `init.js` under `steps` folder

7. Modify `init.js` to the following (using the deployed API Gateway url for the `restaurants_api` environment variable, and use the DynamoDB table you created)

```javascript
let initialized = false

const init = async () => {
  if (initialized) {
    return
  }

  process.env.restaurants_api   = "https://xxx.execute-api.eu-west-1.amazonaws.com/dev/restaurants"
  process.env.restaurants_table = "restaurants-dev-yancui"
  process.env.AWS_REGION        = "eu-west-1"
  
  initialized = true
}

module.exports = {
  init
}
```

8. Open `test_cases/get-index.js`, and require the `init` module at the top

```javascript
const { init } = require('../steps/init')
```

and then modify the `decribe` with a `before` step

```javascript
describe(`When we invoke the GET / endpoint`, () => {
  before(async () => await init())

  it(`Should return the index page with 8 restaurants`, async () => {
```

9. Run the integration test again

`npm run test`

and see that the test passes

```
  When we invoke the GET / endpoint
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (449ms)


  1 passing (467ms)
```

</p></details>

<details>
<summary><b>Add test case for get-restaurants</b></summary><p>

1. Add `get-restaurants.js` under `test_cases`

2. Modify `get-restaurants.js` to the following

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')

describe(`When we invoke the GET /restaurants endpoint`, () => {
  before(async () => await init())

  it(`Should return an array of 8 restaurants`, async () => {
    let res = await when.we_invoke_get_restaurants()

    expect(res.statusCode).to.equal(200)
    expect(res.body).to.have.lengthOf(8)

    for (let restaurant of res.body) {
      expect(restaurant).to.have.property('name')
      expect(restaurant).to.have.property('image')
    }
  })
})
```

3. Open `when.js` and add a `we_invoke_get_restaurants` function

```javascript
const we_invoke_get_restaurants = () => viaHandler({}, 'get-restaurants')
```

and update `module.exports` at the bottom to include this new function

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants
}
```

4. Run the integration test

`npm run test`

and see that the tests pass

```
  When we invoke the GET / endpoint
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (371ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (451ms)


  2 passing (839ms)
```

</p></details>

<details>
<summary><b>Add test case for search-restaurants</b></summary><p>

1. Add `search-restaurants.js` under `test_cases`

2. Modify `search-restaurants.js` to the following

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')

describe(`When we invoke the POST /restaurants/search endpoint with theme 'cartoon'`, () => {
  before(async () => await init())

  it(`Should return an array of 4 restaurants`, async () => {
    let res = await when.we_invoke_search_restaurants('cartoon')

    expect(res.statusCode).to.equal(200)
    expect(res.body).to.have.lengthOf(4)

    for (let restaurant of res.body) {
      expect(restaurant).to.have.property('name')
      expect(restaurant).to.have.property('image')
    }
  })
})
```

3. Open `when.js` and add a `we_invoke_search_restaurants` function

```javascript
const we_invoke_search_restaurants = theme => {
  let event = { 
    body: JSON.stringify({ theme })
  }
  return viaHandler(event, 'search-restaurants')
}
```

and update the `module.exports` at the bottom of the file to include this new function

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants
}
```

4. Run the integration test

`npm run test`

and see that the tests pass

```
  When we invoke the GET / endpoint
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (435ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (440ms)

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
    ✓ Should return an array of 4 restaurants (249ms)


  3 passing (1s)
```

</p></details>

<details>
<summary><b>Add test case for place-order</b></summary><p>

1. Add a file `place-order.js` to `test_cases` folder

2. Open `steps/init.js` and load `order_events_stream` as an environment variable. **Don't forget** to replace `yancui` with your name

```javascript
process.env.order_events_stream = 'orders-dev-yancui'
```

3. To avoid causing the side effect of actually pushing the `order_placed` event into the stream, we'll use the `mock-aws` library to mock out the AWSSDK.

Modify `test_cases/place-order.js` to the following

```javascript
const { expect } = require('chai')
const when = require('../steps/when')
const { init } = require('../steps/init')
const AWS = require('mock-aws')

describe(`When we invoke the POST /orders endpoint`, () => {
  let isEventPublished = false
  let resp

  before(async () => {
    await init()

    AWS.mock('Kinesis', 'putRecord', (req) => {
      isEventPublished = 
        req.StreamName === process.env.order_events_stream &&
        JSON.parse(req.Data).eventType === 'order_placed'

      return {
        promise: async () => {}
      }
    })

    resp = await when.we_invoke_place_order('Fangtasia')
  })

  after(() => AWS.restore('Kinesis', 'putRecord'))

  it(`Should return 200`, async () => {
    expect(resp.statusCode).to.equal(200)
  })

  it(`Should publish a message to Kinesis stream`, async () => {
    expect(isEventPublished).to.be.true
  })
})
```

4. Open `steps/when.js` and add a new `we_invoke_place_order` function

```javascript
const we_invoke_place_order = async (restaurantName) => {
  const body = JSON.stringify({ restaurantName }) 
  return viaHandler({ body }, 'place-order')      
}
```

and add it to the `module.exports` at the bottom

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants,
  we_invoke_place_order
}
```

5. Run integration tests

`npm run test`

and see that all 5 tests are passing

```
  When we invoke the GET / endpoint
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (227ms)

  When we invoke the GET /restaurants endpoint
    ✓ Should return an array of 8 restaurants (1024ms)

  When we invoke the POST /orders endpoint
placing order ID [13fc254e-380e-56fb-8c32-b517f9cf669a] to [Fangtasia]
published 'order_placed' event into Kinesis
    ✓ Should return 200
    ✓ Should publish a message to Kinesis stream

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
    ✓ Should return an array of 4 restaurants (71ms)


  5 passing (1s)
```

</p></details>

<details>
<summary><b>Add test case for notify-restaurant</b></summary><p>

1. Open `steps/init.js` and add `restaurant_notification_topic` as an environment variable. **Don't forget** to change `yancui` to your name.

```javascript
process.env.restaurant_notification_topic = 'restaurants-dev-yancui'
```

2. Open `steps/when.js` and add a `we_invoke_notify_restaurant` function

```javascript
const we_invoke_notify_restaurant = async (...events) => {
  return viaHandler(toKinesisEvent(events), 'notify-restaurant')
}
```

and then add a `toKinesisEvent` function

```javascript
const toKinesisEvent = events => {
  const records = events.map(event => {
    const data = Buffer.from(JSON.stringify(event)).toString('base64')
    return {
      "eventID": "shardId-000000000000:49545115243490985018280067714973144582180062593244200961",
      "eventVersion": "1.0",
      "kinesis": {
        "approximateArrivalTimestamp": 1428537600,
        "partitionKey": "partitionKey-3",
        "data": data,
        "kinesisSchemaVersion": "1.0",
        "sequenceNumber": "49545115243490985018280067714973144582180062593244200961"
      },
      "invokeIdentityArn": "arn:aws:iam::EXAMPLE",
      "eventName": "aws:kinesis:record",
      "eventSourceARN": "arn:aws:kinesis:EXAMPLE",
      "eventSource": "aws:kinesis",
      "awsRegion": "us-east-1"
    }
  })

  return {
    Records: records
  }
}
```

and add the `we_invoke_notify_restaurant` function to the `module.exports` at the bottom

```javascript
module.exports = {
  we_invoke_get_index,
  we_invoke_get_restaurants,
  we_invoke_search_restaurants,
  we_invoke_place_order,
  we_invoke_notify_restaurant
}
```

3. Add a file `notify-restaurant.js` to the `test_cases` folder

4. Modify `test_cases/notify-restaurant.js` to the following

```javascript
const { expect } = require('chai')
const { init } = require('../steps/init')
const when = require('../steps/when')
const AWS = require('mock-aws')
const chance = require('chance').Chance()

describe(`When we invoke the notify-restaurant function`, () => {
  let isEventPublished = false
  let isNotified = false

  before(async () => {
    await init()

    AWS.mock('Kinesis', 'putRecord', (req) => {
      isEventPublished = 
        req.StreamName === process.env.order_events_stream &&
        JSON.parse(req.Data).eventType === 'restaurant_notified'

      return {
        promise: async () => {}
      }
    })

    AWS.mock('SNS', 'publish', (req) => {
      isNotified = 
        req.TopicArn === process.env.restaurant_notification_topic &&
        JSON.parse(req.Message).eventType === 'order_placed'

      return {
        promise: async () => {}
      }
    })

    const event = {
      orderId: chance.guid(),
      userEmail: chance.email(),
      restaurantName: 'Fangtasia',
      eventType: 'order_placed'
    }
    await when.we_invoke_notify_restaurant(event)
  })

  after(() => {
    AWS.restore('Kinesis', 'putRecord')
    AWS.restore('SNS', 'publish')
  })

  it(`Should publish message to SNS`, async () => {
    expect(isNotified).to.be.true
  })

  it(`Should publish event to Kinesis`, async () => {
    expect(isEventPublished).to.be.true
  })
})
```

5. Run integration tests

`npm run test`

and see that the new test is failing

```
1) When we invoke the notify-restaurant function
      "before all" hook:
   TypeError: Cannot read property 'body' of undefined
```

because our `notify-restaurant` doesn't return any response, because it doesn't need to.

6. Open `steps/when.js` and reaplce the `viaHandler` function with the following

```javascript
const viaHandler = async (event, functionName) => {
  const handler = require(`${APP_ROOT}/functions/${functionName}`).handler
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

7. Rerun the integration tests

`npm run test`

and see that all tests are passing now

```
  When we invoke the GET / endpoint
invoking via handler function get-index
loading index.html...
loaded
    ✓ Should return the index page with 8 restaurants (204ms)

  When we invoke the GET /restaurants endpoint
invoking via handler function get-restaurants
    ✓ Should return an array of 8 restaurants (385ms)

  When we invoke the notify-restaurant function
invoking via handler function notify-restaurant
notified restaurant [Fangtasia] of order [4a6058f2-14e8-5a61-9b23-c3e980fa04e7]
published 'restaurant_notified' event to Kinesis
    ✓ Should publish message to SNS
    ✓ Should publish event to Kinesis

  When we invoke the POST /orders endpoint
invoking via handler function place-order
placing order ID [680c6239-1c97-569e-b78d-a1d95e339a6f] to [Fangtasia]
published 'order_placed' event into Kinesis
    ✓ Should return 200
    ✓ Should publish a message to Kinesis stream

  When we invoke the POST /restaurants/search endpoint with theme 'cartoon'
invoking via handler function search-restaurants
    ✓ Should return an array of 4 restaurants (66ms)


  7 passing (689ms)
```

</p></details>

<details>
<summary><b>What other test cases would you add?</b></summary><p>

</p></details>
