# Module 2: Clone demo project

<details>
<summary><b>Clone and deploy the demo project</b></summary><p>

1. Fork this [repo](https://github.com/theburningmonk/production-ready-serverless-workshop-devternity-demo)

and clone it on your laptop

2. Open the repo in your IDE

3. Run `npm install`

4. Open `serverless.yml` and replace all instances of `yancui` with your name. For example, change service to `workshop-` followed by your name. You should fine a total of 5 instances of `yancui` in the `serverless.yml`.

For this demo, we'll simulate sending a push notification via SNS. To keep things simple, we'll deliver the notification by email instead. 

In the `serverless.yml`, go to ln 127 and replace `<INSERT EMAIL HERE>` with your work email.

5. Deploy the demo project

`npm run sls -- deploy`

you should see console output like the following

```
Serverless: Packaging service...
Serverless: Excluding development dependencies...
AWS Pseudo Parameters
AWS Pseudo Parameter: Resources::IamRoleLambdaExecution::Properties::RoleName::Fn::Join::1::2 Replaced AWS::Region with ${AWS::Region}
AWS Pseudo Parameter: Resources::GetDashindexLambdaFunction::Properties::Environment::Variables::restaurants_api::Fn::Join::1::2 Replaced AWS::Region with ${AWS::Region}
AWS Pseudo Parameter: Resources::GetDashindexLambdaFunction::Properties::Environment::Variables::orders_api::Fn::Join::1::2 Replaced AWS::Region with ${AWS::Region}
AWS Pseudo Parameter: Outputs::ServiceEndpoint::Value::Fn::Join::1::2 Replaced AWS::Region with ${AWS::Region}
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service .zip file to S3 (7.53 MB)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
.........................................
Serverless: Stack update finished...
Service Information
service: workshop-yancui
stage: dev
region: eu-west-1
stack: workshop-yancui-dev
api keys:
  None
endpoints:
  GET - https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/
  GET - https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/restaurants
  POST - https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/restaurants/search
  POST - https://7md1iyjlxf.execute-api.eu-west-1.amazonaws.com/dev/orders
functions:
  get-index: workshop-yancui-dev-get-index
  get-restaurants: workshop-yancui-dev-get-restaurants
  search-restaurants: workshop-yancui-dev-search-restaurants
  place-order: workshop-yancui-dev-place-order
  notify-restaurant: workshop-yancui-dev-notify-restaurant
```

Open the first `GET` endpoint in the browser, you should see something like this

![](/images/mod02-001.png)

</p></details>

<details>
<summary><b>Seed the DynamoDB table</b></summary><p>

1. Open `seed-restaurants.js` and replace the only instance of `yancui` with your name. This should match what you used in the `serverless.yml` in the previous step.

2. Run the script

`STAGE=dev REGION=eu-west-1 node seed-restaurants.js`

and go to the landing page again, you should now see some restaurants

![](/images/mod02-002.png)

</p></details>

<details>
<summary><b>Subscribe your email</b></summary><p>

1. Check your inbox, you should have received an email from SNS, like this

![](/images/mod02-003.png)

2. Click on the `Confirm subscription` link

![](/images/mod02-004.png)

</p></details>

<details>
<summary><b>Explore the demo project</b></summary><p>

1. Go to the landing page, and search for `cartoon`, click `Find Restaurants`

2. Click on `Fancy Eats` to place an order

![](/images/mod02-005.png)

3. Check your inbox, you should have received a notification email about the order

![](/images/mod02-006.png)

</p></details>

Congratulations! You have deployed a serverless project with:

* a `GET /` endpoint that returns the landing page HTML
* a `GET /restaurants` endpoint that returns a default list of restaurants
* a `POST /restaurants/search` endpoint to search restaurants by theme
* a `POST /orders` endpoint to place an order and publishes an `order_placed` event into a Kinesis stream
* a `notify-restaurant` restaurant function that reacts to the `order_placed` event and publishes a push notification to SNS

This is how things look from a high level:

![](/images/mod02-007.png)

But, this project is not worthy of production yet, because:

* there are no tests at all
* it's logging with `console.log`
* there's no way to disable debug logging in production
* we can't correlate log messages across multiple functions
* we don't have visibility into what's happening inside the function to debug performance issues
* all the functions share the same IAM role, so they all have too much permission