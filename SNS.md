## If you also want to send an SNS message

Create SNS topic:

```
aws sns create-topic --name someoneBarked
```

Subscribe (note `region` and `accountID`):

```
aws sns subscribe `
  --topic-arn arn:aws:sns:region:accountID:someoneBarked `
  --protocol email `
  --notification-endpoint example@example.com
```

Amazon will send a confirmation email to the email address.

Update `publishBark.js`:
Note: you won't need to npm install `aws-sdk`
Note: substitute accountID

* Top:

```
var AWS = require('aws-sdk');
var sns = new AWS.SNS();
```

* After the graphqlClient.mutate() then:

```
sns.publish(
  {
    Subject: 'Someone barked',
    Message: message,
    TopicArn: `arn:aws:sns:${process.env.AWS_REGION}:accountID:someoneBarked`,
  },
  function (err, data) {
    if (err) {
      console.error(
        'Unable to send message. Error JSON:',
        JSON.stringify(err, null, 2),
      );
    } else {
      console.log(
        'Results from sending message: ',
        JSON.stringify(data, null, 2),
      );
    }
  },
);
```

Rezip:

```
zip -r publishBark.zip publishBark.js node_modules
```

Update lambda function:

```
aws lambda update-function-code `
--function-name publishBark `
--zip-file fileb://publishBark.zip
```