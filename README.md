# AWS AppSync + Lambda + DynamoDB

Connect AppSync to Lambda and DynamoDB so that a change in DynamoDB triggers an AppSync GraphQL subscription notification.

# Setup

## Create DynamoDB table with stream enabled
```
aws dynamodb create-table `
    --table-name IncomingBarks `
    --attribute-definitions "AttributeName=id,AttributeType=S" "AttributeName=timestamp,AttributeType=S" `
    --key-schema "AttributeName=id,KeyType=HASH" "AttributeName=timestamp,KeyType=RANGE" `
    --provisioned-throughput "ReadCapacityUnits=5,WriteCapacityUnits=5" `
    --stream-specification "StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES"
```

Output should include this. Note the `region` and `accountID`:
```
 "LatestStreamArn": "arn:aws:dynamodb:region:accountID:table/IncomingBarks/stream/timestamp
```

## Create BarkLambdaRole

Create file `trust-relationship.json` with:

```
{
   "Version": "2012-10-17",
   "Statement": [
     {
       "Effect": "Allow",
       "Principal": {
         "Service": "lambda.amazonaws.com"
       },
       "Action": "sts:AssumeRole"
     }
   ]
 }
```

Create role:

```
aws iam create-role --role-name BarkLambdaRole `
    --path "/service-role/" `
    --assume-role-policy-document file://trust-relationship.json
```

Create file `role-policy.json` (note region and accountID) with:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:region:accountID:function:publishBark*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:region:accountID:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:DescribeStream",
                "dynamodb:GetRecords",
                "dynamodb:GetShardIterator",
                "dynamodb:ListStreams"
            ],
            "Resource": "arn:aws:dynamodb:region:accountID:table/IncomingBarks/stream/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "sns:Publish"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```
_Note: Permission to publish SNS may or may not be required depending on Lambda function_

Update role policy for previously created role:

```
aws iam put-role-policy --role-name BarkLambdaRole `
    --policy-name BarkLambdaRolePolicy `
    --policy-document file://role-policy.json
```

Check:

```
aws iam get-role --role-name BarkLambdaRole
aws iam get-role-policy --role-name BarkLambdaRole --policy-name BarkLambdaRolePolicy
```

## Create AppSync API

* Open `AWS AppSync console`
* Choose `Create API`
* Choose `Create with wizard`
* Model name: `Bark`
* Fields:
  * `id` -> ID
  * `timestamp` -> String
  * `message` -> String
* API Name: `Bark App`

## Create pass-through mutation

* Go to `Data Sources`
* `Create new Data Source`
* Name: `RealTimePassThrough`
* Data source type: `None`
* `Create`

Add a mutation
* Go to `Schema`
* Add this mutation:
```
notifyBark(id: ID!, timestamp: String, message: String): Bark
  @aws_iam
```
* Add this subscription:
```
onNotifyBark(id: ID, timestamp: String, message: String): Bark
	@aws_subscribe(mutations: ["notifyBark"])
```
* Change the Bark type permission
```
type Bark @aws_api_key
@aws_iam {
	id: ID!
	timestamp: String
	message: String
}
```

* `Save Schema`

* Attach the `RealTimePassThrough` data source to the `notifyBark` mutation resolver
  * Under `Configure the request mapping template` set this:
```
{
  "version": "2017-02-28",
  "payload": $util.toJson($context.arguments)
}
```
  * Under `Configure the response mapping template` set this:
```
$util.toJson($context.result)
```

* `Save Resolver`

* Go to `Settings` and note the API ID
* Update `role-policy.json` to this (note `region`, `accountID`, `API_ID`):
```
{
  "Version": "2012-10-17",
  "Statement": [
      {
          "Effect": "Allow",
          "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:region:accountID:function:publishBark*"
      },
      {
          "Effect": "Allow",
          "Action": [
              "logs:CreateLogGroup",
              "logs:CreateLogStream",
              "logs:PutLogEvents"
          ],
            "Resource": "arn:aws:logs:region:accountID:*"
      },
      {
          "Effect": "Allow",
          "Action": [
              "dynamodb:DescribeStream",
              "dynamodb:GetRecords",
              "dynamodb:GetShardIterator",
              "dynamodb:ListStreams"
          ],
          "Resource": "arn:aws:dynamodb:region:accountID:table/IncomingBarks/stream/*"
      },
      {
        "Effect": "Allow",
        "Action": "appsync:GraphQL",
        "Resource": "arn:aws:appsync:region:accountID:apis/API_ID/*"
      },
      {
          "Effect": "Allow",
          "Action": [
              "sns:Publish"
          ],
          "Resource": [
              "*"
          ]
      }
  ]
}
```

* Update role policy
```
aws iam put-role-policy --role-name BarkLambdaRole `
--policy-name BarkLambdaRolePolicy `
--policy-document file://role-policy.json
```

* In Settings -> Additional authorization providers, add:
  * `AWS Identity and Access Management (IAM)`
* `Save`

* For improved debugging, enable Logs
  * Settings -> Logging
    * `Enable Logs`
    * `Include verbose content`
    * `Field resolver log level` -> `All`
    * `Create or use an existing role` -> `New role`

## Subscribe to onNotifyBark
* Go to `Queries` and subscribe
```
subscription sub1 {
  onNotifyBark {
    id,
    timestamp,
    message
  }
}
```

## Create Lambda function

```
npm install aws-appsync cross-fetch graphql-tag
```

Filename: `publishBark.js`

```
'use strict';
const appsync = require('aws-appsync');
const gql = require('graphql-tag');
require('cross-fetch/polyfill');

const graphqlClient = new appsync.AWSAppSyncClient({
  url:
    'https://path.appsync-region.amazonaws.com/graphql', // get from Settings -> API URL
  region: process.env.AWS_REGION,
  auth: {
    type: 'AWS_IAM',
    credentials: {
      accessKeyId: process.env.AWS_ACCESS_KEY_ID,
      secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
      sessionToken: process.env.AWS_SESSION_TOKEN,
    },
  },
  disableOffline: true,
});

// it matters what you return here. return all id, timestamp, and message, otherwise, subscriber of onNotifyBark won't receive missing fields.
const mutation = gql`
  mutation mutation1($id: ID!, $timestamp: String, $message: String) {
    notifyBark(id: $id, timestamp: $timestamp, message: $message) {
      id
      timestamp
      message
    }
  }
`;

exports.handler = (event, context, callback) => {
  event.Records.forEach(record => {
    console.log('Stream record: ', JSON.stringify(record, null, 2));

    if (record.eventName == 'INSERT') {
      var id = record.dynamodb.NewImage.id.S;
      var timestamp = record.dynamodb.NewImage.timestamp.S;

      if (!record.dynamodb.NewImage.message) {
        callback(
          null,
          `Error processing ${id}. message is undefined.`,
        );
        return;
      }

      var message = record.dynamodb.NewImage.message.S;

      graphqlClient
        .mutate({
          mutation,
          variables: {
            id,
            timestamp,
            message,
          },
        })
        .then(() => {
          // do something else?
        });
    }
  });

  callback(
    null,
    `Successfully processed ${event.Records.length} records.`,
  );
};

```

* zip it all up:

```
zip -r publishBark.zip publishBark.js node_modules
```

* Get role ARN:

```
aws iam get-role --role-name BarkLambdaRole
```

* Create Lambda function (note region, {roleARN})

```
aws lambda create-function `
    --region us-west-2 `
    --function-name publishBark `
    --zip-file fileb://publishBark.zip `
    --role {roleARN} `
    --handler publishBark.handler `
    --timeout 5 `
    --runtime nodejs12.x
```

* If we ever need to update function code

```
aws lambda update-function-code `
--function-name publishBark `
--zip-file fileb://publishBark.zip
```

## Test Lambda function

Create `payload.json`:

```
{
    "Records": [
        {
            "eventID": "7de3041dd709b024af6f29e4fa13d34c",
            "eventName": "INSERT",
            "eventVersion": "1.1",
            "eventSource": "aws:dynamodb",
            "awsRegion": "us-west-2",
            "dynamodb": {
                "ApproximateCreationDateTime": 1479499740,
                "Keys": {
                  "id": {
                      "S": "123456789"
                  },
                  "timestamp": {
                      "S": "2016-11-18:12:09:36"
                  }
                },
                "NewImage": {
                  "id": {
                      "S": "123456789"
                  },
                  "timestamp": {
                      "S": "2016-11-18:12:09:36"
                  },
                  "message": {
                      "S": "Hello world!!!"
                  }
                },
                "SequenceNumber": "13021600000000001596893679",
                "SizeBytes": 112,
                "StreamViewType": "NEW_IMAGE"
            },
            "eventSourceARN": "arn:aws:dynamodb:us-east-1:123456789012:table/IncomingBarks/stream/2016-11-16T20:42:48.104"
        }
    ]
}
```

Execute test:

```
aws lambda invoke  --function-name publishBark --payload file://payload.json output.txt
```

* Check the subscripton listener in AWS console to see if it received notification

## Create trigger

* Get IncomingBarks stream LatestStreamArn

```
aws dynamodb describe-table --table-name IncomingBarks
```

* Find `LatestStreamArn` from the result above
* Create trigger (note `{streamARN}`)

```
aws lambda create-event-source-mapping `
    --region us-west-2 `
    --function-name publishBark `
    --event-source {streamARN}  `
    --batch-size 1 `
    --starting-position TRIM_HORIZON
```

## Test by adding entry to database
```
aws dynamodb put-item `
--table-name IncomingBarks `
--item 'id={S="123456789"},timestamp={S="2020-11-18:14:32:17"},message={S="Hello from the database"}'
```


## Resources:
* https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.Lambda.Tutorial.html
* https://aws.amazon.com/premiumsupport/knowledge-center/appsync-notify-subscribers-real-time/
* https://cloudonaut.io/calling-appsync-graphql-from-lambda/