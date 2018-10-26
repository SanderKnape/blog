+++
title = "Comparing AWS SAM with the Serverless framework"
author = "Sander Knape"
date = 2018-02-22T20:39:02+02:00
draft = false
tags = ["aws", "cloudformation", "dynamodb", "lambda"]
categories = []
+++
Serverless applications are everywhere these days. Having been introduced some years ago with the introduction of AWS Lambda, today serverless is much more then Function as a Service (FaaS). AWS is even starting to use the term in their products: at AWS re:Invent 2017 "Aurora Serverless" was introduced, a fully managed RDMS database.  

How do you build such applications? Given that you properly like the Infrastructure as Code mindset as much as I do, the question is how to properly specify serverless applications provisioned in AWS in code. Two main options are out there: the Serverless Framework and AWS SAM. Both frameworks allow you to make it easier to build serverless applications.  

In this blog post I'm going to put that to the test. Using both the [serverless framework](https://serverless.com) and [AWS SAM local](https://github.com/awslabs/aws-sam-local), I will build and deploy an application. All source code can be found in my [GitHub repository](https://github.com/SanderKnape/comparing-serverless-and-sam), and using the instructions in the [README](https://github.com/SanderKnape/comparing-serverless-and-sam/blob/master/README.md) and in this blog post you can provision the application in your own AWS account.  

The serverless application we will construct and deploy is architected as follows;  

![](/images/serverless_application_architecture.png)  

One of the goals is to test different Lambda event sources, so the architecture exists of a few different types of components. The overall flow is as follows;

1.  Through a PUT request coming into API Gateway, a Lambda function is invoked that puts an item into DynamoDB (a GET request can be used to view the items in DynamoDB, also through a Lambda function)
2.  A Lambda function is attached to the DynamoDB stream. This Lambda function generates an image and puts it into S3
3.  Through the `ObjectCreated` event of S3, a Lambda function is invoked that notifies an SNS topic. This SNS topic then sends an e-mail to notify that the image was created

I'm going to assume some at least basic experience with the tools, so be sure to check out the getting started pages for both the [Serverless Framework](https://serverless.com/framework/docs/getting-started/) and [AWS SAM](https://github.com/awslabs/serverless-application-model/blob/master/HOWTO.md).

# Requirements

What we're testing here is;

*   As mentioned: invoking Lambda functions through different events types;
    *   API Gateway (2x)
    *   DynamoDB Stream
    *   S3 ObjectCreated
*   Dynamic environment variables for Lambda;
    *   The DynamoDB stream ARN
    *   The S3 bucket name
    *   The SNS topic ARN
*   IAM permissions: giving each resource permissions to other resources following the principle of least privilege (only exactly those permissions required and no more)
*   Automatically deploying our Lambda functions and API changes: what if we need to run a script before packaging, e.g. `npm install`?

# The contenders

The **[Serverless Framework](https://serverless.com)** is a framework that makes it easy to write event-driven functions for a myriad of providers, including AWS, Google Cloud, Kubeless and more. For each provider, a series of events can be configured to invoke the function. The framework is [open source](https://github.com/serverless/serverless) and receives updates regularly.  

The **[AWS Serverless Application Model (SAM)](https://github.com/awslabs/serverless-application-model)** is an abstraction layer in front of CloudFormation that makes it easy to write serverless applications in AWS. There is support for three different resource types: Lambda, DynamoDB and API Gateway. Using **[SAM Local](https://github.com/awslabs/aws-sam-local)**, Lambda and API Gateway can be run locally through the use of Docker containers.  

Both frameworks have in common that they generate CloudFormation. In other words: they both abstract CloudFormation so that you need to write less code to build serverless applications (in the case of SAM) and to deploy Lambda functions (for both SAM and Serverless). The biggest difference is that Serverless is written to deploy FaaS (Function as a Service) functions to different providers. SAM on the other hand is an abstraction layer specifically for AWS using not only FaaS but also DynamoDB for storage and API Gateway for creating a serverless HTTP endpoint.  

Another difference is that SAM Local allows you to run Lambda functions locally and to spin up an API Gateway locally. This makes it easier to develop and test Lambda functions without deploying them to AWS. With the Serverless framework you can also invoke Lambda functions from the command line, but only if they are deployed to AWS and available through API Gateway.

# The stacks

I constructed the above architecture both using the Serverless Framework and with SAM. The YAML files can be found in my [GitHub repository](https://github.com/SanderKnape/comparing-serverless-and-sam): [serverless.yaml](https://github.com/SanderKnape/comparing-serverless-and-sam/blob/master/serverless.yaml) and [template.yaml](https://github.com/SanderKnape/comparing-serverless-and-sam/blob/master/template.yaml).  

The obvious difference between the two stacks is the length of both files: the Serverless yaml is significantly larger. This is largely because of a missing feature with IAM permissions (see the IAM comparison below). We also need to specify a little more metadata for the serverless framework to get it running.  

If you're already familiar with CloudFormation, you will notice that the SAM yaml is very much alike CloudFormation. AWS SAM is written as an extension of CloudFormation, using transformations (see line 2) to transform the syntax to valid CloudFormation. The Serverless yaml file is more typical YAML with some metadata on top. This metadata is close to [CloudFormation parameters](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html), though in a more readable way.  

After cloning the GitHub repository, you can deploy the stack using both frameworks. The frameworks create all resources with different names, so both stacks can be deployed at the same time. Make sure you have your IAM credentials set up properly before deploying the stacks. Follow the installation instructions in the repository to install the stacks. In general, the commands you need to run are as follows (more steps are required to deploy the example stacks; check out the repository for those).  

**Serverless**  

Use the following command to deploy the stack using the Serverless framework:  

`serverless deploy -v`  

The `-v` verbose flag is added so that we can track the status of the deployment - basically a command line view on top of the CloudFormation events.  

Remove the stack with the following command:  

`serverless remove`  

**SAM**  

With SAM, an S3 bucket must first be created to which the deployment artifacts are used. Therefore, first create an S3 bucket and then run the following commands to deploy the stack:  

`sam package --template-file template.yaml --s3-bucket [**your_s3_bucket]** --output-template-file package.yaml`  

`sam deploy --template-file package.yaml --stack-name serverless-application --capabilities CAPABILITY_IAM`  

Both command are equal to their [CloudFormation counterparts](http://docs.aws.amazon.com/cli/latest/reference/cloudformation/index.html#cli-aws-cloudformation).  

To remove the stack, run the following command:  

`aws cloudformation delete-stack --stack-name serverless-application`  

As you can see, the Serverless framework commands are definitely easier to remember and use. The `deploy` command both packages and deploys the stack, whereas with SAM two different commands are required to be ran. Also, an S3 bucket must be created first for SAM and more parameters need to be specified in the commands. The Serverless framework generates the S3 bucket itself and picks its own stack name and package name. If you already have an S3 bucket, you can specify this in the yaml file using the provider.deploymentBucket key ([see docs](https://serverless.com/framework/docs/providers/aws/guide/serverless.yml/)). The stack name can only be partly specified by using the --stage parameter in the deploy command. The stack name is always serverless-application-\[stage\], with production as the default stage.  

Another benefit of the Serverless CLI commands is the generated output. The -v flag (verbose) outputs any updates to the stack in the console. With SAM, creating or updating a stack doesn't show any other info than "Waiting for stack create/update to complete". In addition, when removing a stack, the Serverless CLI waits until the stack is destroyed. The default Cloudformation CLI immediately exits, giving you no info whether the stack is successfully deleted or not.  

Next, let's dive more into the specific requirements I listed in the introduction and compare the differences between Serverless and SAM.

## Specifying events

Both frameworks specify events on which to invoke Lambda functions in pretty much the same way. For example, an API Gateway event looks as follows:  

**Serverless**

```yaml
events:
  - http:
      path: /items
      method: GET
```

**SAM**

```yaml
Events:
  GetApiEndpoint:
    Type: Api
    Properties:
      Path: /items
      Method: GET
```

For a DynamoDB stream event, the syntax is as follows:  

**Serverless**

```yaml
events:
  - stream:
      type: dynamodb
      batchSize: 10
      startingPosition: TRIM_HORIZON
      arn:
        Fn::GetAtt:
          - DynamoDBTable
          - StreamArn
```

**SAM**

```yaml
Events:
  DynamoDBInsert:
    Type: DynamoDB
    Properties:
      Stream: !GetAtt "DynamoDBTable.StreamArn"
      StartingPosition: TRIM_HORIZON
      BatchSize: 10
```

The Serverless syntax is a little more readable, until [CloudFormation intrinsic functions](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html) come into play. This is a general drawback of the Serverless framework: the shorthand syntax is not supported, requiring us to use multiple lines of code in some situations. For example, the `GetAtt` function as is required for getting the DynamoDB stream ARN.

## Dynamic environment variables

Specifying environment variables is also very similar for both frameworks. An example:  

**Serverless**

```yaml
environment:
  DynamoDBTableName:
    Ref: DynamoDBTable
```

**SAM**

```yaml
Environment:
  Variables:
    DynamoDBTableName: !Ref "DynamoDBTable"
```

For some reason, with SAM you need to specify a `Variables` key within the `Environment` map. This gives room for future expansion of the Environment configuration. Other than that, these properties are very similar and it's easy to reference dynamic CloudFormation resources that are only available after a resource is created. In both cases, the function is only created after the depending resource (such as DynamoDB in this case) is created.

## Least-privilege IAM permissions

Specifying IAM permissions is the first bigger difference between the two services. As mentioned above, IAM is the main reason that the Serverless yaml file is significantly bigger. This is because it is not possible to extend the IAM policies per-function. One function needs permissions to S3, another one to SNS. To specify these permissions, an entire role needs to be created, including the default Lambda managed policy (AWSLambdaBasicExecutionRole) and the AssumeRolePolicyDocument. A [GitHub issue](https://github.com/serverless/serverless/issues/4313) with this feature request already exists. With SAM, it is possible to extend the role of a function with specific policies.  

The Serverless framework does allow you to specify a different default role for each Lambda function on a global level. In addition, it is possible to extend the role for all Lambda functions. This is not possible through SAM.  

One thing I noticed is that when using the DynamoDB event in SAM, a policy is attached with the `Resource` set to wildcard (\*). This means that the Lambda function has the permissions to read all the streams in the account. Since an [event source mapping](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-eventsourcemapping.html) is required to attach the Lambda function to the DynamoDB stream, this isn't technically a security issue. But it's certainly an improvement that could be made. With the Serverless framework, a policy is created with the Resource set specifically to the DynamoDB stream that is configured.

## Local development

One of the drawbacks of using AWS Lambda used to be that the Lambda function could not be invoked locally in an environment similar to the AWS Lambda environment. This is exactly why [SAM Local](https://github.com/awslabs/aws-sam-local) was created as an extension to SAM. The Lambda functions you specify in SAM can now be invoked locally. The function is then run using a [Docker container](https://github.com/lambci/docker-lambda) created by the people behind [LambCI](https://github.com/lambci/docker-lambda).  

In addition, SAM Local can spin up a local API Gateway environment. Any Lambda functions integrated with API Gateway can be invoked through your browser or curl command. This is great for local development.

## In general

Finally, some last thoughts/experiences I had with both frameworks;  

**Serverless**

*   Shorthand intrinsic functions don't work in the Resources section. These shorthand functions are definitely more readable and usable than the longer alternatives, so support for this is certainly welcome.
*   S3 bucket support is rather "magic". For example, take the situation in our stack where we attach a Lambda function to the S3 ObjectCreated event. The S3 bucket is created in the Resources section below. Ideally, we would simply reference that S3 bucket (as we do in the SAM stack) but this is not possible as Serverless [does some magic with naming resources](https://serverless.com/framework/docs/providers/aws/guide/resources#aws-cloudformation-resource-reference). I'm sure this is supposed to make using the framework easier, but to me it only seems like unnecessary complexity.

My main complaint here is that "vanilla CloudFormation" isn't supported by the serverless framework. The magical naming and the fact that shorthand intrinsic functions are not supported make it harder to get started.  

**SAM**

*   I can not use the SimpleTable resource as that can not give me the stream ARN. I have to use the AWS::DynamoDB::Table resource so that I can do a `!GetAtt "DynamoDBTable.StreamArn"`. I created a [feature request](https://github.com/awslabs/serverless-application-model/issues/227) for this.
*   A shortcut command for packing/deploying a stack would certainly be nice.
*   Some metadata file (or within the stack) would be nice, for example for CI/CD commands to run or the S3 bucket to use.

When running your Lambda function in AWS, an IAM role is attached giving it specific permissions to AWS resources. When running a Lambda function locally with SAM, the access keys on your machine are used. If somehow STS could be used to actually use the Lambda permissions, local development would be one step closer to the actual environment in which Lambda runs for production. To be honest: this problem isn't specific to SAM but a more general issue with developing Lambda functions. SAM would be a great place though to add functionality to make this easier.  

For both frameworks, what I'm missing is some more CI/CD capabilities. Before deploying the Lambda functions, a `npm install` must be ran to fetch the dependencies. It is not possible to specify this within either framework. For SAM, there is a [Makefile example](https://github.com/awslabs/aws-sam-local/blob/develop/samples/python-with-packages/Makefile) where these pre-package scripts are being executed. It would be great if a formal, built-in specification would exist for the frameworks.  

My experience with SAM was definitely a lot more positive, perhaps especially because I am already experienced with writing CloudFormation.

# Conclusion

In this blog post, I gave a small introduction to using the Serverless Framework and AWS SAM. The tools can both be used to deploy serverless application, though as mentioned both tools have their specific use cases. I hope this blog post will help you in choosing the right tool for your specific use case.
