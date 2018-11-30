# Module 7: SSM Parameter Store

## Use SSM parameter store to configure resource names, URLs, etc.

**Goal:** Configurations are managed in SSM parameter store

<details>
<summary><b>Add dev configurations</b></summary><p>

1. Go to EC2 console

2. Go to `Parameter Store` (bottom left)

3. Click `Create Parameter`

4. Use the name `/xxx/dev/table_name` and **replace** `xxx` with `workshop-` followed by your name, e.g. `workshop-yancui`

![](/images/mod07-001.png)

5. Click `Create Parameter`

6. Repeat step 3-5 to create another `/xxx/dev/stream_name` parameter with the value `orders-dev-` followed by your name, e.g. `orders-dev-yancui`

7. Repeat step 3-5 to create another `/xxx/dev/restaurant_topic_name` parameter with the value `restaurants-dev-` followed by your name, e.g. `restaurants-dev-yancui`

8. Repeat step 3-5 to create another `/xxx/dev/url` parameter with the root URL for your deployed API without the ending `/`, e.g. `https://exun14zd2h.execute-api.eu-west-1.amazonaws.com/dev`

</p></details>

<details>
<summary><b>Use SSM to parameterise serverless.yml</b></summary><p>

1. Replace the `TableName` for the `restaurantsTable` DynamoDB table (in the `resources` section of your `serverless.yml`) with `${ssm:/${self:service}/${self:provider.stage}/table_name}`

```yml
resources:
  Resources:
    restaurantsTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${ssm:/${self:service}/${self:provider.stage}/table_name}
        AttributeDefinitions:
          - AttributeName: name
            AttributeType: S
        KeySchema:
          - AttributeName: name
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
```

2. Replace the `Name` for the `orderEventsStream` Kinesis stream (in the `resources` section of your `serverless.yml`) with `${ssm:/${self:service}/${self:provider.stage}/stream_name}`

```yml
orderEventsStream:
  Type: AWS::Kinesis::Stream
  Properties: 
    Name: ${ssm:/${self:service}/${self:provider.stage}/stream_name}
    ShardCount: 1
```

3. Replace the `DisplayName` and `TopicName` for the `restaurantNotificationTopic` SNS topic (in the `resources` section of your `serverless.yml`) with `${ssm:/${self:service}/${self:provider.stage}/restaurant_topic_name}`

```yml
restaurantNotificationTopic:
  Type: AWS::SNS::Topic
  Properties: 
    DisplayName: ${ssm:/${self:service}/${self:provider.stage}/restaurant_topic_name}
    TopicName: ${ssm:/${self:service}/${self:provider.stage}/restaurant_topic_name}
```

4. Deploy the project

`npm run sls -- deploy`

5. Run the acceptance test

`npm run acceptance`

to make sure everything is still working as before.

</p></details>

<details>
<summary><b>Use SSM to parameterise seed-restaurants script</b></summary><p>

1. Open the `seed-restaurants.js` script, go to line 51, where we defined the `getTableName` function

```javascript
const getTableName = async () => {
  return `restaurants-${STAGE}-yancui`
}
```

replace this function with the following

```javascript
const getTableName = async () => {
  console.log('getting table name...')
  const req = {
    Name: `/xxx/${STAGE}/table_name`
  }
  const ssmResp = await ssm.getParameter(req).promise()
  return ssmResp.Parameter.Value
}
```

**REMINDER**: don't forget to replace `xxx` in the SSM parameter path with what you used in your `serverless.yml`

2. Rerun the script

`STAGE=dev REGION=eu-west-1 node seed-restaurants.js`

and go to DynamoDB console to see that the newly created stage-specific table is now populated

</p></details>

<details>
<summary><b>Use SSM to parameterise tests</b></summary><p>

1. Replace the `steps/init.js` with the following

**NOTE**: replace `workshop-yancui` with your service name

```javascript
const _ = require('lodash')
const { promisify } = require('util')
const awscred = require('awscred')
const { REGION, STAGE } = process.env
const AWS = require('aws-sdk')
AWS.config.region = REGION
const SSM = new AWS.SSM()

let initialized = false

const getParameters = async (keys) => {
  const prefix = `/workshop-yancui/${STAGE}/`
  const req = {
    Names: keys.map(key => `${prefix}${key}`)
  }
  const resp = await SSM.getParameters(req).promise()
  return _.reduce(resp.Parameters, function(obj, param) {
    obj[param.Name.substr(prefix.length)] = param.Value
    return obj
   }, {})
}

const init = async () => {
  if (initialized) {
    return
  }

  const params = await getParameters([
    'table_name',
    'stream_name',
    'restaurant_topic_name',
    'url'
  ])

  console.log('SSM params loaded')

  process.env.TEST_ROOT                     = params.url
  process.env.orders_api                    = `${params.url}/orders`
  process.env.restaurants_api               = `${params.url}/restaurants`
  process.env.restaurants_table             = params.table_name
  process.env.AWS_REGION                    = REGION
  process.env.order_events_stream           = params.stream_name
  process.env.restaurant_notification_topic = params.restaurant_topic_name
  
  initialized = true
}

module.exports = {
  init
}
```

2. Rerun the integration tests

`STAGE=dev REGION=eu-west-1 npm run test`

and see that all the tests are passing

3. Rerun the acceptance tests

`STAGE=dev REGION=eu-west-1 npm run acceptance`

4. Commit and push your changes to see that they're still passing on CodePipeline too

</p></details>