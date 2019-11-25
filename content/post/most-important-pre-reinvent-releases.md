+++
title = "Roundup of the most important pre-re:Invent 2019 releases - so far"
author = "Sander Knape"
date = 2019-11-25T11:04:24+02:00
draft = false
tags = ["aws", "kubernetes", "eks", "dynamodb", "sqs", "fargate", "cloudformation", "cloudwatch", "iam"]
categories = []
+++
The most exciting time of the year for AWS Enthusiasts is upon us. In exactly seven days, AWS re:Invent 2019 will kick off and everyone is excited to see what great features will be released and announced this time around. 

This year especially though, many new features are already released the weeks leading up to re:Invent. If you haven't been paying attention, it was easy to much some great new announcements. Therefore, in this blog post, a roundup of the (in my opinion) most important AWS releases in the past few weeks.

## EKS managed node groups

Kubernetes is without a doubt the most popular (DevOps) tool in recent years. It is no secret that the Google Cloud offering, [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) (GKE), is still the most mature and advanced managed Kubernetes offering. For a long time, the [Amazon Elastic Kubernetes Service](https://aws.amazon.com/eks/) (EKS) would only manage the control plane. Management of the worker nodes - including security updates, version updates, node draining, e.t.c. - was still up to the customer.

That has changed now with the release of [EKS managed node groups](https://aws.amazon.com/blogs/containers/eks-managed-node-groups/). You can now attach node groups to your EKS control plane, very similar to how you would attach node groups to your Google Cloud GKE clusters. AWS fully takes care of the management of these node groups. You can attach multiple node groups to a single cluster, for example to add different EC2 instance types to your cluster. 

Given the speed at which more competitors offer better managed Kubernetes offerings, this was an update long overdue. It's great to see AWS is taking the first steps towards managing EKS worker nodes for its customers.

If you are considering migrating to EKS managed node groups, I would advice to wait just another 1.5 week or so. Announced a [solid two years ago](https://aws.amazon.com/blogs/aws/aws-fargate/) already, it looks like the release of Fargate for EKS is finally around the corner. This feature is on the top of the [AWS container roadmap](https://github.com/aws/containers-roadmap/projects/1), and a [new IAM policy](https://twitter.com/__steele/status/1197746212406906880?s=19) has already surfaced. This may be one of the most important re:Invent announcements this year.

Fargate for EKS promises to bring an even more advanced managed Kubernetes offering to the landscape. Given AWS has been working on this for at least two years already, I know my expectations are high.

## IAM sessions tags

One of the most interesting challenges that I love to tackle is how to provide development teams as much autonomy as possible so that they can do their work without relying on external factors (such as a cloud or platform team). Earlier this year I blogged about this subject, presenting [five methods to give developers autonomy in AWS](https://sanderknape.com/2019/07/five-ways-enable-developer-autonomy-aws/). 

It just became easier to provide developers fine-grained IAM permissions to AWS. Most companies allow developers to login to the AWS Console through a system such as Active Directory or Okta. You can now [pass in user attributes (session tags)](https://aws.amazon.com/blogs/aws/new-for-identity-federation-use-employee-attributes-for-access-control-in-aws/) from these systems. What that means is that you can pass variables such as cost centers or team names to AWS, and use these variables in IAM policies. In the following example (copied from the announcement blog post), a variable `CostCenter` is used to give a user permissions to Start/Stop only specific EC2 instances:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [ "ec2:DescribeInstances"],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": ["ec2:StartInstances","ec2:StopInstances"],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "ec2:ResourceTag/CostCenter": "${aws:PrincipalTag/CostCenter}"
                }
            }
        }
    ]
}
```
*Code copied from the [announcement blog post](https://aws.amazon.com/blogs/aws/new-for-identity-federation-use-employee-attributes-for-access-control-in-aws/)*

This is extremely powerful and makes it much easier to give developers permission to only resources that they own. You can, for example, use prefixes or tags to give developers access to only their own secrets in Secrets Manager. You can also give developers access to view (and edit) data only in DynamoDB tables or S3 buckets that they own.

I do believe that infrastructural changes to AWS should only be made through code using Infrastructure as Code (IaC) tools such as Terraform, CloudFormation or the AWS CDK. Only give your developers some basic management permissions to make it easier for them to own their resources in AWS. All other changes should definitely be made through IaC in a CI/CD pipeline.

## Lambda support for SQS FIFO queues

While this may look like just a small change, [Lambda's supporting FIFO SQS queues](https://aws.amazon.com/about-aws/whats-new/2019/11/aws-lambda-supports-amazon-sqs-fifo-event-source/) is a pretty big release for anyone who's ever worked with [event-driven architecture](https://microservices.io/patterns/data/event-driven-architecture.html). Such architectures are a powerful way for creating loosely-coupled systems where different services communicate through asynchronous events or messages. If your system handles orders for an e-commerce websites, other systems may want to subscribe to these changes so that they can handle their own logic based on these events.

However, the ordering of these events is often very important. If ordering is not guaranteed, a `finishOrder` event may be picked up by a different system before the `addProductToCart` event that was emitted right before it. Additional logic to tackle these situations may be required, leading to more work for developments teams. SQS FIFO queues guarantee ordering, and thus are often preferred for systems where receiving events in the correct order is important.

While AWS Lambda is often a convenient method for running code in AWS, the lack of support for FIFO queues often meant development teams had to resort to other solutions such as spinning up a virtual machine or a Docker container when working with an SQS FIFO queue. These solutions typically require more maintainance compared to a Serverless solution such as AWS Lambda. Therefore, it's great that we can now finally consider AWS Lambda when processing messages from an SQS FIFO queue.

## DynamoDB global replicas

DynamoDB is a robust, high-performant service for storing data. When your company operates in different parts of the world, it is very convenient to spin up a global DynamoDB table that replicates the data to different AWS regions.

But what if today you need the table only in a single region, while later as your company expands to other parts of the world you want to replicate the data to different AWS regions? You can now [convert a single-region table to a global table](https://aws.amazon.com/blogs/aws/new-convert-your-single-region-amazon-dynamodb-tables-to-global-tables/). Before, you would have to create a new table (with the "global" flag turned on) and synchronize the data from the old table to this new table. 

For companies that are growing to different parts of the world, this is a small but oh-so convenient new feature. Instead of paying for a more expensive global table while you don't need it yet, you can now turn it on when it is actually required.

## CloudFormation drift detection for stack sets

CloudFormation is a great tool for IaC. And so is Terraform. Both tools have their own advantages, and many blogs and articles have been written comparing the two. 

One area that Terraform excels in is how it can keep the *known* state (what Terraform remembers to be the actual state) and the *actual* state synchronized. If another process (e.g. human interaction or some other form of automation) changes the state of a resource, Terraform would effortlessly pick up this inconsistency and update its known state while planning new changes.

CloudFormation, however, would just assume that the actual state is still equal to the known state. This can lead to unintended side-effects, ranging from annoying to disastrous. Now, however, CloudFormation can [detect drift as part of planning changes](https://aws.amazon.com/about-aws/whats-new/2019/11/cloudformation-announces-drift-detection-support-in-stackSets/). This is an important step in bringing this side of CloudFormation more on part with Terraform.

Unfortunately, however, CloudFormation does not yet synchronize the actual state to the known state automatically. This will still have to be done manually. In addition, drift detection is not yet supported for every resource and every property. I'm extemely happy though that the CloudFormation team is working on new features to improve the usability of CloudFormation, and therefore I consider this to be an important release in the right direction.

## CloudWatch Servicelens

CloudWatch (AWS's monitoring and logging service) has never been a very feature-rich service - at least compared to other services that focus fully on monitoring and logging. Therefore many companies set up [other](https://www.datadoghq.com/) [tools](https://www.splunk.com/) for central monitoring and logging. Slowly though, CloudWatch is adding more features and the gap between CloudWatch and these other tools is getting smaller. This means that the percentage of companies who can "just" use CloudWatch grows, and the need to invest in yet-another-tool decreases. In other words: getting started with AWS becomes easier.

Last week AWS announced [CloudWatch ServiceLens](https://aws.amazon.com/about-aws/whats-new/2019/11/announcing-amazon-cloudwatch-servicelens/). ServiceLens essentially provides a dashboard that displays a service map of the components of your application. This makes it much easier to get an overview of the behavior of your application, and eases debugging applications that consist of (many) different components. ServiceLens is therefore especially suited for Serverless applications, where you may combine for example a set of Lambda functions, SQS queues and DynamoDB tables to provide functionality to your customers.

[![CloudWatch ServiceLens](/images/cloudwatch_servicelens.png)](/images/cloudwatch_servicelens.png)
*Image taken from [the AWS documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ServiceLens.html)*

This functionality certainly isn't new: both [Datadog](https://www.datadoghq.com/) and [Epsagon](https://epsagon.com/) have a very similar feature set. It's great though that we can now at get started within AWS, and move to a more advanced tool set later.

## Conclusion

With AWS re:Invent 2019 around the corner, it's an exciting time for any developer who works with AWS. In case you missed some of the recent announcements leading up to re:Invent, I hope this post was of help to you. If you want to stay up to date more with AWS releases, check out their [What's new page](https://aws.amazon.com/new/).