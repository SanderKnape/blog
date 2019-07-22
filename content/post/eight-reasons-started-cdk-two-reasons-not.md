+++
title = "Eight reasons to get started with the AWS CDK - and two reasons not to"
author = "Sander Knape"
date = 2019-07-17T13:47:02+02:00
draft = false
tags = ["kubernetes", "gitlab", "automation", "ci/cd"]
categories = []
+++

Mention both CloudFormation and Terraform as alternatives.

Refer to previous blog. Important things to know to understand this blog:

* The CDK generates CloudFormation, and that CloudFormation is deployed to AWS.
* The CDK is written in Typescript and all the examples in this blog are written in Typescript. However, the CDK supports multiple languages such as Python, Java and C#.

# Eight great things about the CDK

## 1. Sane defaults

Explain how constructs how sane defaults.

## 2. Abstractions with Constructs

Now explain how you build your own constructs.



One of the biggest selling point of the CDK.

## 3. Strong typing and autocompletion

## 4. Grants

The thing I like the least about writing Infrastructure as Code? Tying everything together. It's easy to configure a few Lambda functions, some SNS topics, queues and databases. But then I have to create numerous IAM roles that give least-privilege permissions to all required connections. It slows me down, and I'm sure it's a reason for many people to start using shortcuts. Before you know it, Lambda functions have admin permissions and you have just deployed some intense security risks.

Pro tip: _don't_ use shortcuts. Never accept a quick win for a long-term security issue.

Another pro tip: Use the CDK! In many cases the CDK will create least-privileges IAM roles and policies for you automatically. Look at this simple example of subscribing an SQS queue to an SNS topic:

```typescript
const queue = new sqs.Queue(this, 'queue');

const topic = new sns.Topic(this, 'topic');

topic.addSubscription(new subs.SqsSubscription(queue));
```

Running `cdk synth` on this will show you that a `AWS::SQS::QueuePolicy` CloudFormation resource has been created. This includes a least-privilege `PolicyStatement`:

```yaml
Statement:
  - Action: sqs:SendMessage # the single permissions required for the subscription
    Condition:
        ArnEquals:
            aws:SourceArn:
                Ref: topic69831491 # only the specific SNS topic gets access
    Effect: Allow
    Principal:
        Service: sns.amazonaws.com
    Resource:
        Fn::GetAtt:
          - queue276F7297
          - Arn
```

There are still situations of course where you have to explicitely grant the permissions yourself. This is also much easier with the CDK. Check out the following example where a Lambda function gets permissions to write to a DynamoDB table:

```typescript
const fn = new lambda.Function(...);
const table = new dynamodb.Table(...);

table.grantWriteData(fn);
```

This gives the Lambda function Put, Update and Delete access to the table. If you want to give only specific permissions, you can also grant access to specific IAM actions:

```typescript
table.grant(fn, 'dynamodb:PutItem');
```

This is one of the greatest examples in my opinion where the CDK takes care of some of the heavy lifting for you. It's simple and elegant, but oh so comfortable.

## 5. You can still directly write CloudFormation resources

## 6. You can always read and overwrite the CloudFormation properties

<!-- ## 7. A synergy of imperative and declarative practices

A big plus of tools like CloudFormation and Terraform is that they are declarative. They _declare_ (describe) what the situation should look like: the desired state. Whatever processes that desired state then looks at the current state, and figures out what steps must be taken to get there.

Programming languages like Typescript are imperative. Using a set of statements you write a sequence of steps that must be taken to reach some goal. 

The CDK

You therefore still have the advantage of the declarative nature of CloudFormation. AWS still calculates the diff to get in the desired state, and manages that store for you. -->

## 8. Built-in Custom Resources

One of the biggest complaints I (and many others) have about CloudFormation is that it does not provide full AWS coverage. New services and features are still often released without CloudFormation support. It is not uncommon to see such support added in Terraform much more quickly. But there is actually a good reason for this: the Terraform AWS provider is [open source](https://github.com/terraform-providers/terraform-provider-aws) so whenever you miss a feature, you can add it yourself.

CloudFormation however has a method for easily extending the specification with [Custom Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html). These allow you to add a new CloudFormation resource which invokes a Lambda function. Using the AWS CLI within that function you can then make the changes that are not yet supported within CloudFormation. I've written a blog post on [creating a CloudFormation custom resource](https://sanderknape.com/2017/08/custom-cloudformation-resource-example-codedeploy/) if you want to know more.

Through the CDK it is possible to deploy custom resources to extend the capabilities of CloudFormation. When you want to execute a single API call you can use the [custom resource construct](https://github.com/aws/aws-cdk/tree/master/packages/%40aws-cdk/custom-resources). The CDK will then deploy a Lambda function together with your other resources. For more complex use cases you can also write a larger custom resource yourself. One example already built-in in the CDK is [automatic DNS-validation for ACM certificates](https://github.com/aws/aws-cdk/tree/master/packages/%40aws-cdk/aws-certificatemanager#automatic-dns-validated-certificates-using-route53). When you request a TLS certificate this way, the CDK deploys a Lambda function that automatically inserts the expected DNS entries in Route53. I've used this construct a few times already and it saved me SO much time.

There have been some attempts already at centralizing custom resources in several GitHub repositories. I would however suggest to start centralizing these custom resources in the CDK. There is now finally a way to make a contribution for extending CloudFormation in an official location. You can help yourself but also others by extending the capabilities of CloudFormation this way!

## 9. Bonus

Whilst the above eight reasons should be enough already to convince you that the CDK is a great tool, there is more. Some quick additional reasons to work with the CDK are;

* The CDK supports multiple languages, so you can use the language that you are already familiar with.
* The CDK works great with [AWS SAM](https://github.com/awslabs/aws-sam-cli) which you can use to run and debug your Lambda functions locally. I explain how to set this up in my [previous blogpost](https://sanderknape.com/2019/05/building-serverless-applications-aws-cdk/#local-development).
* The CDK is in active development and the core members are very active on [GitHub](https://github.com/aws/aws-cdk), [Gitter](https://gitter.im/awslabs/aws-cdk) and the AWS Developers Slack workspace.

# Two challenges with the CDK



