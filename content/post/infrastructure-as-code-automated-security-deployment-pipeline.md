+++
title = "Use Infrastructure as Code for automated security in the deployment pipeline"
author = "Sander Knape"
date = 2017-05-01T19:25:02+02:00
draft = false
tags = ["automation", "aws", "cloudformation", "cloudsploit", "lambda", "security"]
categories = []
+++
Infrastructure as Code (IaC) is a very powerful concept. The idea is that you put all infrastructure resources - networks, subnets, load balancers, firewalls and so on - in code. You then deploy your infrastructure the same way application developers deploy their code: through a continuous integration / continuous deployment (CI/CD) pipeline. Other benefits already reaped by application developers that become available are code linting, automated testing and an audit trail of your changes if combined with a version control system. The most well known IaC tools are [Terraform](https://www.terraform.io/) (supports many different services) and [CloudFormation](https://aws.amazon.com/cloudformation) (specifically for the AWS cloud).  

Earlier this week I was reading a blog post about [discovering security improvements](https://summitroute.com/blog/2017/05/30/free_tools_for_auditing_the_security_of_an_aws_account/) in your AWS account. The post compares different tools that scan your infrastructure for security vulnerabilities and some of those tools will take action and fix/remove the offending resource. Such tools are vital when running production load and should definitely be enabled from day one to keep people with bad intentions out of your infrastructure - or to be alerted of mistakes that can easily be made.  

A disadvantage of these tools though is that they come after the fact: they report/fix vulnerabilities _already in your infrastructure_. This brings us to another benefit of IaC: it allows you to scan your infrastructure code _before deploying it to your environment_. This prevents you from rolling out vulnerabilities into your environment instead of fixing them. Additionally, it aligns very well with the [early feedback / shift left](https://en.wikipedia.org/wiki/Shift_left_testing) approach. The earlier in your process that you catch errors (whether it is a security vulnerability or a syntax error), the quicker your development process.

# Security in the deployment pipeline

It seems that security is finally catching up with CI/CD pipelines. For years, developers already benefit from the agility that CI/CD brings them but security is always this "extra" entity that lives outside from those pipelines. If we can put everything in code, why not include security audits there as well?  

Last week I was at the [DevOpCon](https://devopsconference.de/) conference and multiple talks discussed how security has become part of their delivery pipelines. For example, Dutch bank ING works together closely with auditors to automatically generate their security reports within their pipelines. This saves the company a lot of work since they don't need to manually fill out large forms anymore. In addition, VolksWagen Financial Services mentioned how they run a penetration test on their Lambda functions within their deployment pipeline before deployment to the production environment.  

Security in CI/CD pipelines of course goes much further than scanning infrastructure code. We already use our acceptance/staging environments for automated acceptance tests, performance tests and the likes. This is where can add security tests as well and, as ING is already doing, generate the reports for auditors and make the lives for both developers and security teams. I believe it's up to security teams to catch up with their developers by joining forces and to investigate how security can become part of the new way of automation.  

Let's dive into a specific tool to give an example of how easy it is to set up an infrastructure security scan in our deployment pipeline.

# A practical example with CloudSploit

There are not yet many tools out there that can scan infrastructure code: further evidence that this mindset is still rather new. One tool that I did get quite some experience with in the past few months is [CloudSploit](https://cloudsploit.com/). I wish I could mention additional tools but a search didn't return me any results.  

A downside of CloudSploit is that you have to pay a monthly fee to get access to their CloudFormation scan API. Automation is therefore probably less suited for your hobby projects, but it's definitely worth considering for the company you are working for. In the rest of this blog post, we're going to set up a CI/CD pipeline that scans our CloudFormation template before deploying it to production.  

The tools we use are [AWS CodeCommit](https://aws.amazon.com/codecommit/), [AWS CodePipeline](https://aws.amazon.com/codepipeline), [AWS Lambda](https://aws.amazon.com/lambda) and, of course, CloudFormation and CloudSploit. With CodePipeline it is very easy to deploy CloudFormation code and its lack of features (for now, hopefully) make it easy to get started with it. It hooks in really well to CodeCommit, but you can certainly follow along if your code is in a [GitHub repository](http://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-four-stage-pipeline.html).  

The CloudSploit API endpoint we will use can be found over at the [CloudSploit documentation](http://docs.cloudsploit.apiary.io/#reference/scanning-apis/cloudformation-scan).

## Setting up the pipeline

![](/images/codepipeline.png)I am going to assume some basic knowledge with CodePipeline. The pipeline that we will build looks as the image on right. Let's set up the pipeline step by step. Open up the AWS CodePipeline console and create a new pipeline. Give it an appropriate name and configure each step as follows.

### Source

The first step will hook on to the source repository we will use. I'm using CodeCommit but feel free to hook on a GitHub repository if you want. Create a new repository: this is where we are going to host the CloudFormation code that we'll scan before deployment.

### Build

We are not going to build anything, so choose "No Build" here. After we created the pipeline, we'll add the step for running the security scan before the deployment.

### Deploy

The final stage is the deploy action category. Select AWS CloudFormation as the deployment provider. Select `MyApp` as your Input Artifact (this is the default name for the output artifact in the first stage). Setup the CloudFormation deployment configuration as follows:  

![](/images/codepipeline_deployment.png)  

As for you IAM role (the last configuration option above): you need a role that can create an EC2 security group. For real production systems, be sure to use the least-privilege principle to only give access to the resources it needs to touch.

### Service role

This is where you specify the IAM role that CodePipeline will use while it executes. Create a role that can create and update CloudFormation stacks and invoke Lambda functions.

### The security scan Lambda function

Next we are going to setup Lambda function that will scan the CloudFormation code in the VCS repository. In between the "Source" and "Deploy" steps, add a new stage and use the "Invoke" action category. Again, select `MyApp` as your Input Artifact.  

Create a new Python 3 Lambda function and use [this source code from my GitHub repository](https://github.com/SanderKnape/cloudformation-security-scan/blob/master/lambda.py). Also, be sure to add your CloudSploit access key and secret as environment variables. Name them `api_key` and `secret`.  

The function should be pretty self explanatory. Just keep in mind that I tried to keep it as minimal as possible so it has no error handling whatsoever. What is basically does is grab a "template.yaml" from the VCS root we configured and send the contents to CloudSploit so we can run the security scan.  

With everything in place, we can create a CloudFormation template and deploy it!

# Testing our template.yaml

Let's set up a simple CloudFormation template with a firewall rule that [is considered a bad practice](https://aws.amazon.com/articles/1233/): we open up port 22 (SSH) to the entire world. A better way is to open this port only within our internal network or otherwise only to trusted, public IP addresses. The template is as follows:

```yaml
SecurityGroup:
  Type: "AWS::EC2::SecurityGroup"
  Properties:
    GroupDescription: "Open up port 22 to the world"
    SecurityGroupIngress:
      - IpProtocol: "tcp"
        FromPort: 22
        ToPort: 22
        CidrIp: "0.0.0.0/0"
```

Commit this file in the root of your project as `template.yaml` to your CodeCommit or GitHub repository. CodePipeline will automatically be triggered. Let's see what happens!  

![](/images/codepipeline_failed.png)  

It looks like the function failed to let's figure out what went wrong. This is actually a rather annoying shortcoming of CodePipeline: if you click on the "details" link, you'll get send to the CloudWatch logs of the Lambda function. Now that we just ran the function, we know we can find the proper logs by opening the top link. However, if you come back later to this function and the Lambda function will have run again in the meanwhile, it might be hard to find the error.  

Anyway, open up your Lambda logs and you should find the following error:

```
Error 'Security group is being created in EC2 Classic. Only VPCs should be used.' for resource 'SecurityGroup'
```

Whoops: we forgot to specify a VPC ID. Without that we would create a legacy security group. Let's change the template so it looks as follows:

```yaml
SecurityGroup:
  Type: "AWS::EC2::SecurityGroup"
  Properties:
    GroupDescription: "Open up port 22 to the world"
    VpcId: "vpc-xxx"
    SecurityGroupIngress:
      - IpProtocol: "tcp"
        FromPort: 22
        ToPort: 22
        CidrIp: "0.0.0.0/0"
```

Replace the VpcID with the real ID of your VPC (otherwise, you won't be able to properly deploy this security group). Commit the updated template and wait for CodePipeline to pick up the change. Next, we get the following message from CloudSploit:

```
Error 'Security group ingress exposes a non-web port globally: 22' for resource 'SecurityGroup'
```

Looks like we almost deployed a security vulnerability! Let's fix this by specifying an IP address for our security group:

```yaml
SecurityGroup:
  Type: "AWS::EC2::SecurityGroup"
  Properties:
    GroupDescription: "Open up port 22 to the world"
    VpcId: "vpc-xxx"
    SecurityGroupIngress:
      - IpProtocol: "tcp"
        FromPort: 22
        ToPort: 22
        CidrIp: "8.8.8.8/32"
```

We now only allow the public DNS servers from Google to connect to port 22 through our security group (which, granted, doesn't make much sense: more logical is for example the public IP address from your office). Commit this code, wait for CodePipeline and you will see your code is finally safely deployed. Great success!

# Conclusion

In this blog post we looked at using CloudSploit to scan infrastructure as code before deploying it to our environment. Security in the deployment pipeline is still a rather new concept and mindset, but I believe it's a welcome addition to our toolset for automating the entire life cycle of both infrastructure and applications.
