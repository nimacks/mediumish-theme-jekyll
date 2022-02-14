---
layout: post
title:  "Triggering builds on-demand with AWS Amplify Hosting"
author: sam
categories: [ aws, amplify ]
tags: [yellow]
image: assets/images/build.jpg
description: "Triggering builds on-demand with AWS Amplify Hosting"
featured: true
hidden: false
toc: true
comments: true
#rating: 4.5
---

In this post we’ll explore an alternative way to set up continuous delivery for our full stack application. Out of the box AWS Amplify provides us with a fully integrated CI/CD system which builds an application after each code push. Now, this works great  in most workflows. However what if we want our builds to run on a custom schedule.

<img src="https://nimacks.s3.us-west-2.amazonaws.com/img/ondemand.png" alt="drawing" width="765"/>

We’ll setup an incoming Webhook in the AWS Amplify console. Next we’ll create a lambda function that will run on a predefined schedule using CloudWatch Events.

### Disable Automatic builds

The first thing we want to do is to disable automatic builds.In the Amplify Console click on your app and navigate to general settings.

Select  the branch to use for on-demand builds. In the Action drop down select `Disable auto build.`

### Create an incoming Web-hook

Select the build settings of your app. Under Incoming webhooks click `Create webhook`. A new webhook will be created which we will use to trigger builds in our lambda function. The webhook command should look similar to this.

```python
curl -X POST -d {} "https://webhooks.amplify.us-west-2.amazonaws.com/prod/webhooks?id=caa8&operation=startbuild" 
-H "Content-Type:application/json"
```

### Configure your lambda function

```python
const axios = require("axios");
const url =
  "https://webhooks.amplify.us-west-2.amazonaws.com/prod/webhooks?id=caa1db03-428e-44df-bb89-5d45523d91a1&token=IfBcE1WfuVEuggbQlIPRlmDesSLAk2VSpHp74ARur8&operation=startbuild";
let response;

exports.lambdaHandler = async (event, context) => {
  try {
    const headers = {
      "Content-Type": "application/json"
    };

    const data = await axios.post(url, {}, {headers: headers});
    console.log(data);
    response = {
      statusCode: 200,
      body: JSON.stringify({
        message: "successfully triggerd build"
      }),
    };
  } catch (err) {
    console.log(err);
    return err;
  }

  return response;
};
```

## Configure the CloudWatch Event

```bash
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  triggerbuild

  Sample SAM Template for triggerbuild
  

Globals:
  Function:
    Timeout: 3

Resources:
  TriggerBuildFunction:
    Type: AWS::Serverless::Function 
    Properties:
      CodeUri: trigger-build/
      Handler: app.lambdaHandler
      Runtime: nodejs14.x
      Architectures:
        - x86_64
      Events:
        TriggerBuildSchdule:
          Type: Schedule
          Properties:
            Schedule: rate(10 minutes)
            Name: trigger-build-schedule
            Description: Amplify on demand build schedule
            Enabled: True

Outputs:
 TriggerBuildFunction:
    Description: "Hello World Lambda Function ARN"
    Value: !GetAtt TriggerBuildFunction.Arn
  TriggerBuildFunctionIamRole:
    Description: "Implicit IAM Role created for Hello World function"
    Value: !GetAtt TriggerBuildFunctionRole.Arn
```

If you’re using AWS Sam to setup your application you can test this locally by running the following command 

```bash
sam local invoke
Invoking app.lambdaHandler (nodejs14.x)
START RequestId: cc2f6eca-3c7d-4f84-9db5-4cedc3e01688 Version: $LATEST
} } SendMessageResponse: 
{ ResponseMetadata: [Object],
 SendMessageResult: [Object] }d=caa1db03-428e-44df-bb89-5d45523d91a1&token=IfBcE1WfuVEuggbQlIPRlmDesSLAk2VSpHp74ARur8&operation=startbuild',
END RequestId: cc2f6eca-3c7d-4f84-9db5-4cedc3e01688
REPORT RequestId: cc2f6eca-3c7d-4f84-9db5-4cedc3e01688  Init Duration: 1.09 ms  Duration: 1828.62 ms    Billed Duration: 1829 ms        Memory Size: 128 MB     Max Memory Used: 128 MB
{"statusCode":200,"body":"{\"message\":\"successfully triggerd build\"}"}%
```

That’s it, a new build will be triggered every 10mins according to the schedule we defined. Of course you can change this to suit your particular use case.

Hope this helps.