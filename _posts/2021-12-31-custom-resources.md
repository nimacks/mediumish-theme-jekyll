---
layout: post
title:  "Managing Custom Resources with CloudFormation"
author: sam
categories: [ aws ]
tags: [yellow]
image: assets/images/cloudformation.jpg
description: "Managing Custom Resources with CloudFormation"
featured: true
hidden: false
toc: true
rating: 4.5
---

One of the core tenets of building applications in the cloud is infrastructure as code. The ability to script and automate the creation of cloud resources increases productivity.

AWS CloudFormation is a service that helps you model and set up your cloud resources so that you can spend less time managing those resources and focus your time on your applications.

CloudFormation supports a host of services like Alexa, Amazon MQ, EC2 and many more. However, not all AWS services support CloudFormation. In addition, there might be use cases  not inherently supported by CloudFormation. This is where custom resources come in handy.  

Custom Resources is a feature of CloudFormation which allows you to hook into the pre-defined CloudFormation events. These events are  `Create`, `Update`, `Delete`.  Using a simple lambda function and AWS SDK APIs, you can provision and manage the lifecycle of any AWS Service which doesn't support CloudFormation out of the box.

### How does it work
Creating custom resources is pretty straightforward.

<img src="https://nimacks.s3-us-west-2.amazonaws.com/img/Custom+Resources+with+Cloudformation.png" alt="drawing" width="765"/>

<img src="https://nimacks.s3-us-west-2.amazonaws.com/img/Custom+Resources+with+Cloudformation+-+components.png" alt="drawing" width="765"/>

### 1. Designing the resource definition
In the CloudFormation template you'll define a new resource of type ```Custom::YourThing```. The only required property is `ServiceToken`. An example is provided below.

```yaml
CanaryDeployer:
     Type: Custom::CanaryDeployer
     DependsOn:
      CanaryDeployerFunction
     Properties:
      ServiceToken:
        Fn::GetAtt:
          - CanaryDeployerFunction
          - Arn
      Bucket: 
      Key: 
      Region:
        Ref: AWS::Region
      CanaryDeployerRoleName:
        Fn::GetAtt:
          - CanaryDeployerExecutionRole
          - Arn
```

### 2. Request
```json
{
    "RequestType": "Create",
    "ServiceToken": "arn:aws:lambda:us-west-2:xx:function:sam-dev-froyo-imitation-johndoe-1VHIWMMG9Q3PO",
    "ResponseURL": "https://cloudformation-custom-resource-response-uswest2.s3-us-west-2.amazonaws.com/",
    "StackId": "arn:aws:cloudformation:us-west-2:xxx:stack/sam-dev-froyo-imitation/ss-7da9-11e9-9992-ss",
    "RequestId": "06153a82-166c-4e8b-a1d7-ss",
    "LogicalResourceId": "LambdaDeployer",
    "ResourceType": "Custom::LambdaDeployer",
    "ResourceProperties": {
        "ServiceToken": "arn:aws:lambda:us-west-2:xxx:function:sam-dev-froyo-imitation-DeployerFun-829",
        "Bucket": "sam-dev-us-west-2-froyo-xxx",
        "HostingCanaryDeployerRoleName": "arn:aws:iam::xxx:role/sam-dev-froyo-imitation-DeployerExe-Y9266E4RI2FM",
        "Region": "us-west-2",
        "Key": "samtoolkit-Lambda-1.0/xxx/xx-3c0b-4bc4-9ac8-xx-lambda.zip"
    }
}
```
### 3. Writing the resource code 
When you get round to writing the resource code, here are some best practices:
- Separate complex logic from your Create/Update/Delete functions
- Log the ResponseUrl from the Request (or the whole request) first
- Everything goes in a try block (or other exception handler)
- Make sure all code paths lead to a response
- Keep track of time

```java
public String handleRequest(final Map<String, Object> handlerEvent, final Context context) {

        System.out.println("Receiving custom resource event:  " + gson.toJson(handlerEvent));


        DeploymentRequest deploymentRequest = new DeploymentRequest(handlerEvent, context);
        System.out.println("deployment request:  " + gson.toJson(deploymentRequest));

        final JsonObject responseData = new JsonObject();

        try {
            if (deploymentRequest.getRequestType().equalsIgnoreCase(REQUEST_TYPE_CREATE)) {
                createResources(deploymentRequest);
                sendResponse(deploymentRequest, STATUS_SUCCESS, responseData);


            } else if (deploymentRequest.getRequestType().equalsIgnoreCase(REQUEST_TYPE_UPDATE)) {
                updateRegionResources(deploymentRequest);
                sendResponse(deploymentRequest, STATUS_SUCCESS, responseData);

            } else if (deploymentRequest.getRequestType().equalsIgnoreCase(REQUEST_TYPE_DELETE)) {

                deleteRegionResources(deploymentRequest);
                sendResponse(deploymentRequest, STATUS_SUCCESS, responseData);

            }
        } catch (Exception e) {
            System.out.println("Failed to create canary resources : " + e.getMessage());
            e.printStackTrace();
            sendResponse(deploymentRequest, STATUS_FAILED, responseData);
        }

        return null;
    }
```

### 4. Response

```java
/**
     * Send a response to CloudFormation regarding progress in creating resource.
     */

    private Object sendResponse(final DeploymentRequest deploymentRequest,
                                final String responseStatus,
                                final JsonObject responseData) {

        System.out.println("Sending Response: START");

        String responseUrl = deploymentRequest.getResponseUrl();
        deploymentRequest.getContext().getLogger().log("ResponseURL: " + responseUrl);

        URL url;
        try {
            url = new URL(responseUrl);
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();
            connection.setDoOutput(true);
            connection.setRequestMethod("PUT");

            JsonObject responseBody = new JsonObject();
            responseBody.addProperty("Status", responseStatus);
            responseBody.addProperty("PhysicalResourceId", deploymentRequest.getStackId());
            responseBody.add("StackId", gson.toJsonTree(deploymentRequest.getStackId()));
            responseBody.add("RequestId", gson.toJsonTree(deploymentRequest.getRequestId()));
            responseBody.add("LogicalResourceId", gson.toJsonTree(deploymentRequest.getLogicalResourceId()));
            responseBody.add("Data", responseData);

            OutputStreamWriter response = new OutputStreamWriter(connection.getOutputStream());
            response.write(responseBody.toString());
            response.close();

            System.out.println("Sending Response: END" + gson.toJson(responseBody));

            deploymentRequest.getContext().getLogger().log("Response Code: " + connection.getResponseCode());

        } catch (IOException e) {
            System.out.println("An error occurred sending a response");
            e.printStackTrace();
        }

        return null;
    }
```