+++
title = "The hidden challenges of Serverless: from VM to function"
author = "Sander Knape"
date = 2018-08-02T12:29:02+02:00
draft = false
tags = ["dynamodb", "lambda", "aws"]
categories = []
+++
Serverless is a relatively new term. It's a software development paradigm where the entire concept of a "server" is abstracted away from the development process. You essentially only use managed services that handle scaling, and you pay only for what you use. You no longer need to think about operating systems, security patches, scaling configuration and more. All this is handled for you behind the scenes. The most well-known example is AWS Lambda, though as we will see in this blog post, serverless is much more than that.

With a paradigm that promises so much, it's easy to expect too much and get disappointed when you start working with the technology. When you think a service will handle everything for you, expecting to just "upload some code", you're going to have a hard time finding out that there is still much you need to think about.

Don't get me wrong: serverless is a great concept and software development is definitely hastened when using it. It's important though to jump in with the right expectations, which is exactly what I'm hoping to help you with in this post.

I expect that most people who end up here already write software which is deployed to either a container, a virtual machine (VM) or even bare metal. The perspective for this post is therefore the shift from "VM" to serverless. Let's discuss the hidden challenges that you may not expect when diving into serverless technology.

## Is serverless the right solution to your problem?

When diving into serverless technology for the first time to solve a problem, the first question to ask yourself is: is this really the right solution? It's easy to get started with serverless because it's the latest hype, the new thing everybody on reddit is talking about. However, serverless is just another way to solve a problem, and definitely not a silver bullet that can solve every problem.

Learning when to use serverless and when not to is really something that you will learn primarily through experimentation. Begin with small, simple use cases and slowly start experimenting with more complex situations. At some point, you're going to hit a "limit", and find that you can solve a problem better with "old-school" virtual machines.

Of course, there are some good guidelines for when Lambda is or isn't a good solution. Let's dive into a few of these.

### Lambda runtime limit

Lambda is designed for short-running, simple functionalities that scale very horizontally. Instead of running a single process that takes an hour, the idea is to run 3600 functions at once for only a second. A Lambda function can only execute for up to 5 minutes.

Many workloads of course won't fit in this structure. Long-running batch processes can take up to hours or days, and are therefore very hard to translate to Lambda. There are examples where a Lambda function [applies recursion](https://hackernoon.com/write-recursive-aws-lambda-functions-the-right-way-4a4b5ae633b6), and calls itself again when the timeout is starting to approach. In general I wouldn't recommend these solutions as they add complexity and would rather suggest spinning up a container or VM for a longer-running solution. If your process is easy to break up in smaller jobs, give Lambda a try but also keep in mind it might not be the ideal solution.

### Function complexity

Something else to consider is the complexity of your Lambda function. How many business logic do you expect to put in your function? A general best practice is to keep your function small. For example, [NewRelic found that most Lambda functions are very small](https://blog.newrelic.com/2017/11/21/aws-lambda-state-of-serverless/).

"*The bias towards smaller function code size suggests the majority of the New Relic-monitored functions running on AWS Lambda contain relatively few bundled dependencies or extensive business logic, instead pointing to potentially simpler functions. This supports the general best practice around creating small functions designed to perform a single, well-defined task.*"

An application with tens of thousands of lines of code (or more), with many different routes and paths probably doesn't translate very well to a Lambda function. You can definitely give it a try (and let me know the results!), but keep in mind Lambda wasn't designed for this.

### Pricing

Looking at the [Lambda Pricing page](https://aws.amazon.com/lambda/pricing/), you pay for both the amount of invocations and for how long your functions run. The first pricing example on that page specifies a Lambda function that;

*   Is configured with 512MB of memory
*   Executes 3 million times a month
*   Runs for 1 second each time

This function is expected to cost a little over 18 dollars per month. Compare this with 2 EC2 t2.nano instances behind an AWS load balancer. This will cost around 28 dollars per month (20 for the load balancer and 4 for each instance). Those costs are therefore equal to around 4.5 million executions each month for your Lambda function.

4.5 million may sound like an awful lot of invocations, but this number actually translates to only **~1.7 invocations per second**. That suddenly isn't so much anymore.

Of course, running on EC2 instance requires more work as you also have to configure the OS, run regular patch updates, e.t.c. It's therefore a hard comparison to make. However, keep in mind that Lambda isn't as cheap as you might expect at first.

If you want to read more about Lambda pricing, check out this in-depth study recently released about [the economics of serverless](https://www.bbva.com/en/economics-of-serverless/).

## You're going to have to do more than just writing code

Lambda supports a pretty wide variety of languages. There is therefore a big chance that you're already familiar with one of the languages. Running a Lambda function should be breeze, right?

Writing, deploying, running and maintaining your Lambda function actually requires a little more work. Clicking together the function in the AWS Console and changing the code is easy, but it gets harder when you want to get it production ready. To give some examples:

*   You're going to want to setup a deployment pipeline for your function(s), which means that you should somehow specify your Lambda functions in code. Luckily, some great frameworks exist that make this process easier for you. I recently wrote a blog post comparing two of these: [AWS SAM and the Serverless framework](https://sanderknape.com/2018/02/comparing-aws-sam-with-serverless-framework/). You definitely want to invest some time into learning these tools, and picking the one that works best for your use case.
*   An important concept to understand regarding Lambda is [concurrency control](https://docs.aws.amazon.com/lambda/latest/dg/concurrent-executions.html). There is a single limit for the amount of functions that can execute in parallel *per account*. This means that if a single Lambda function that is invoked many times, can cause all your other Lambda functions to break. Thus in theory, you can cause a Denial of Service for all your functions. Use concurrency control for your functions to limit and reserve the number of concurrent invocations per function. And consider increasing the default limit of 1000 through AWS support.
*   Lambda is designed for high concurrency, but what about the upstream dependencies that you are using within your function? Connection pooling with Lambda functions isn't possible as every function creates its own connection. If you are using RDS for example, you have to make sure the database can handle the amount of connections that you expect to make with it.

It's impossible to know from the beginning what you're going to need to think about. The main message here is not to underestimate it, and to take your time to learn the entire lifecycle of your function. Serverless technology is much more than just writing a function. Which brings me to my next point.

## Serverless is more than just AWS Lambda / Function as a Service

Up to now we've primarily been talking about Lambda, but serverless is much more than just "Function as a Service" (FaaS). What about all those queues, load balancers and databases that you are running on a VM? You can migrate those to serverless solutions as well.

When starting to use the non-Lambda serverless technologies, you start seeing that you're mainly clicking a set of lego blocks together to solve a problem. Your job is essentially transforming to more of an architectural role. This means that you need to learn what lego blocks exist and when to use which. In AWS, these lego blocks include but are not limited to:

*   Lambda for your compute needs
*   DynamoDB for your document-storage needs
*   SQS and SNS for publish/subscribe an queueing patterns, creating loosely-coupled and separated architectures
*   API gateway for providing a HTTP (REST) service
*   S3 for file storage
*   Athena for data analysis
*   Cognito for authentication

You're going to have to go through the learning curve for at least most of these lego blocks. Based on your specific use case, you might need to answer some of the following questions for yourself:

*   How does DynamoDB work? Is my data suited for it? What throughput should I use? Should I use autoscaling? What should be the primary and secondary key?
*   How will I deploy Lambda? Will I use serverless framework, AWS SAM, chalice or something else? How do these tools work?
*   How do I setup my SQS queue? Should I use a Dead Letter Queue? How do I deal with messages that end up here?
*   How do I migrate my existing users to Cognito? Does it support MFA? Will it work with my on-prem AD setup?
*   Why is the API Gateway interface so annoying to use? How do I get it in code? How do mapping templates work? Should I use the Proxy setting for Lambda or not?
*   How should I structure my data in S3 for easy processing? Is S3 strongly consistent?
*   How do I get my monitoring, logging, metrics and alerts setup?
*   Is all of this secure? Are my auditors going to like it?
*   How do I get all this stuff in Terraform / CloudFormation / code?

Again: it's all about expectations. Using these services is more efficient than spinning up your own resources on a virtual machine that you need to maintain. But that doesn't mean you're not going to have to do research in the tools you will use.

## Re-think your monitoring, logging, metrics and alerts

Having insight in your serverless application is just as essential as when you maintain the underlying resources yourself. The metrics you will watch are different, but they're still there.

Always check out the AWS documentation for which metrics are most important for a service (e.g. [Lambda](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-functions-metrics.html) or [DynamoDB](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/dynamo-metricscollected.html)). An important metric in general is throttling, as this might cause data loss. When you're reaching the Lambda concurrently limit or the DynamoDB throughput, requests will be throttled and it's up to the client to try again. If that also doesn't work, the request will fail.

Such metrics are pushed to [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/), the managed monitoring solution provided by AWS. To properly set up your logs, metrics and alerts, you're going to need to spend some time in this tool to properly set everything up. This is advice not specific to using serverless technology but for AWS in general, but be sure to invest enough time in it.

If you are already using an external tool such as [Datadog](https://www.datadoghq.com/) or [New Relic](https://newrelic.com/), you will find that these tools can connect to your AWS account and fetch the metrics. If you don't feel comfortable with using CloudWatch or you want to keep using your existing solution, this is an option to consider.

## Developing serverless on your laptop

Since serverless is essentially about putting a bunch of lego blocks together, the question is whether these lego blocks can also run on your laptop. When writing a Lambda function that is for example triggered by an SNS topic, it's nice to run your integration test locally to see if your Lambda function works as expected (in complement to a unit test for which you don't need the topic).

If you run RabbitMQ in the cloud using Docker, it's easy to spin up that Docker image on your laptop for local development. For AWS resources, a solution that works really well is [LocalStack](https://github.com/localstack/localstack). The tool is easy to use as you just need to spin up a Docker container, but be sure to invest some time in it to get it in your development workflow.

You may have an external service that your Lambda function connects with which is available through a local Docker container. If using AWS SAM, this is possible [by specifying the Docker network to use](https://github.com/awslabs/aws-sam-cli#connecting-to-docker-network). If you don't do this, your Lambda function can not connect with your other containers.

It takes a little time with getting set up with running serverless solutions locally. However, if you follow these tips, development is pretty much the same as it was before. Which brings me to a round of non-challenges that you don't have to worry about when starting with serverless.

## Turning it around: the not so hidden challenges

Let's end with some examples of concepts that are *not* so different from "traditional" software development. If you expect challenges in one of the following areas, I'm happy to share that these concepts are very similar to what you're probably already used to.

### Unit testing

[The very first best practice in the AWS documentation](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html) mentions how you should separate your Lambda handler from the core logic. The reason for this is test-ability: through separating your core logic in a separate function/class, you can easily write a unit test for that function/class if you properly inject your dependencies.

Compare this with a typical MVC pattern setup where you also separate your core logic from your controllers. You only unit test the logic and not the controllers. When doing this properly in Lambda, your unit test setup is very comparable to what you're already used to.

### Injecting (secret) configuration

If you're following the [Twelve-Factor app](https://12factor.net/) method for injecting configuration, you'll feel right at home with Lambda. Using environment variables you inject environment-specific configuration including sensitive information. The common method for secrets is to use AWS KMS to decrypt the encrypted secrets in your Lambda function.

Check out the [AWS documentation regarding environment variables](https://docs.aws.amazon.com/lambda/latest/dg/env_variables.html) for more information.

### Debugging with breakpoints

When using AWS SAM, it's still possible [to set breakpoints](https://github.com/awslabs/aws-sam-cli#debugging-applications) when debugging your code.

### Deployments

There are many different ways to deploy an application to a virtual machine. With Lambda, this is relatively simple. You generate a ZIP file, upload it so S3 and create a new Lambda version referencing the ZIP file. If you use [AWS CodeDeploy](https://docs.aws.amazon.com/lambda/latest/dg/automating-updates-to-serverless-apps.html), you can enable blue/green deployments for your function which is very similar to the "traditional" blue/green deployments for applications. If you use a tool such as SAM or the Serverless framework, these details are largely handled for you.

What's very similar from before though is that in general, the process still looks as follows:

*   Build and compile your code
*   Run continuous inspection tools such as linting and unit testing
*   Generate a deployable artifact
*   Deploy the artifact to a staging environment
*   Optional but very much recommended: Run an acceptance/integration/system test on your staging environment
*   Deploy the artifact to the production environment

Don't over-think your deployment pipelines. Apart from the details, the general flow of deployments is still the same as it was before.

## Conclusion

Don't underestimate serverless. I've seen situations where the fully-managed selling point created some unrealistic expectations. You're still going to have to learn many new things, though the learning curve will pay off faster and will prepare you for faster software development in the future. Be sure to properly manage the expectations, whether it's to yourself, your peers or your manager. The shift to managed services essentially forces you towards a more architectural role, which requires a different kind of skill set. And never forget: serverless isn't a silver bullet.

I hope the tips in this post gives you some good context and provides realistic expectations for working with serverless technology. Though much is different, much is still the same as well and in the end, it's still software development. Have fun!
