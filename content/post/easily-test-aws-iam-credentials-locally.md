+++
title = "How to easily test your AWS IAM credentials locally"
author = "Sander Knape"
date = 2018-09-02T18:30:02+02:00
draft = false
tags = ["aws", "iam", "python"]
categories = []
+++
It is still very common to develop an application locally on a laptop/desktop before pushing it to a production-like environment. The local development environment is kept as close as possible to production using technology such as Docker or AWS SAM when working with AWS Lambda. However, when working with AWS resources through Identity and Access Management (IAM) policies, local IAM permissions are typically different from the permissions the application will have in AWS. This inconsistency can cause issues later in the development workflow: an application that fully worked locally can run into errors when ran in AWS if the IAM permission there are different.

In this blog post I first zoom in into the issue a bit more and then explain how I solved this issue for myself using a simple Python utility.

## IAM permissions: the challenge

It is not uncommon to work with admin-like AWS IAM permissions (for a development, testing or staging AWS account, hopefully not production!) in a local development environment. You can login to the AWS account and see and change pretty much every resource. You create STS tokens for local use, using the AWS CLI or the SDK in your applications. If these applications use other AWS resources such as an SQS queue or a DynamoDB table, they have no problem connecting to these resources because the application is using your admin-like permissions. Everything works and is easy, fine and happy.

You then push your application to AWS where it runs as a Lambda function or within an EC2 instance. Of course, you follow security’s best practices and apply the [least-privilege principle](https://en.wikipedia.org/wiki/Principle_of_least_privilege) to all your AWS resources. Your resources only have exactly the permissions it needs to connect to other resources.

This is an annoying inconsistency between your local development environment and the first stage of pushing your application to an actual AWS account. Applying the principle of [early feedback](https://blog.cogent.co/the-importance-of-early-feedback-d0ac3f61cbcc), the sooner you learn your IAM permissions are off, the better.

## My solution

The solution is simple, really: assume the role that your application (Lambda / EC2) is going to assume in AWS and use it while running the application in your local development environment. All that is needed is a simple utility that makes it easy to switch between different IAM roles. As I searched around for possible solutions, my requirements were;

* Make it easy to switch between different IAM roles. When working on a number of Lambda functions, each of these functions might have their own IAM role. It should therefore be easy to switch between the different roles used by the Lambda function.
* An intuitive, easy-to-remember command line interface. I don’t want to copy/paste my role ARN each time or look in my bash history for the correct role when I need to assume a different role. Instead, I prefer to use an alias to easily switch to a previously-configured role.
* Less is more. Just a single utility with a single purpose.

As I wasn’t able to find a tool fulfilling these requirements, I put one together myself. With two simple steps you can use it to easily switch between roles.

### Configure your AssumeRolePolicyDocument

First, you will need to edit the `AssumeRolePolicyDocument` for the role you are going to assume from your local development environment. The following JSON is a default Lambda `AssumeRolePolicyDocument` including an additional line that gives my **development** role permissions to assume this role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com",
        "AWS": "arn:aws:iam::**012345678912**:role/**development**"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Important**: be sure **never** to give permissions such as these to a resource in a production account. This is a huge risk and opens up a simple way to mistakenly change resources in your production account, and opens up the potentials for abusing these permissions to retrieve sensitive data.

### Install and use the `assume` utility

Next, install my [assume](https://github.com/SanderKnape/assume) utility with pip:

```bash
pip install assume
```

Now, say you have two different Lambda functions that both use a different IAM role:

* A `publisher` function using IAM role `arn:aws:iam::012345678912:role/publisher`
* A `subscriber` function using IAM role `arn:aws:iam::012345678912:role/subscriber`

First, make sure to edit the `AssumeRolePolicyDocument` for these roles as described above. You need to give the role or user that you typically login with to have permissions to assume this role.

Next, let’s add these roles to our `assume` configuration:
```bash
assume add publisher --role-arn arn:aws:iam::012345678912:role/publisher

assume add subscriber --role-arn arn:aws:iam::012345678912:role/subscriber
```

You can optionally use the `--profile` flag to use a profile other than the default profile for assuming the role.

Now you can easily switch between the two different roles. For example, run the following command to assume the `publisher` role:

```bash
assume switch subscriber
```

To clear any role and switch back to your default role, run the `clear` command:

```bash
assume clear
```

Check out the readme in the [GitHub repository](https://github.com/SanderKnape/assume) for the other commands.

## Conclusion

Increase your development cycle and receive early feedback regarding IAM permissions. This solution should make it easier to spot IAM permission errors earlier in the development workflow, and will make it easier to implement strict, least-privilege IAM permissions for your AWS resources.

Have you ran into this issue before? Do you use a similar or a completely different solution? I’m interested in other approaches so definitely let me know!
