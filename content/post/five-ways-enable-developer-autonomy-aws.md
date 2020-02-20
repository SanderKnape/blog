+++
title = "Five ways to enable developer autonomy in AWS"
author = "Sander Knape"
date = 2019-07-23T12:00:00+02:00
draft = false
tags = ["aws", "aws config", "s3", "iam", "platform"]
categories = []
+++

It hasn't been that long since it was normal to request compute capacity at some operations department within your organization. In fact, it's probably still pretty common in some organizations. With the move to virtualization and especially the cloud, this process of course has changed dramatically for the good. Not only compute capacity for applications, but also resources such as databases, queues, load balancers and storage are now available virtually unlimited.

Of course, fully opening up the permissions to create these resources in the cloud poses some risks. Costs may inflate and there are many different security risks involved. Examples are accidentally putting (sensitive) data in public S3 buckets, creating public databases with default admin passwords, or opening up the SSH port to the world.

It is not uncommon for organizations to limit these risks by having every resource request validated by a specific team. This may be an ops team, a DevOps team, a security team, architects, or a platform team. Name it whatever you want. What it comes down to is that someone is in the middle of the development lifecycle for developers who want to build features. However, the gap between dev and ops has severely shrunk in the last few years. How is a queue different these days from an additional function in your code? Especially if that queue is provisioned through Infrastructure as Code, it's all code in the end.

I believe that it is possible these days to grant developers full autonomy in the Cloud. A team - I choose to call it a platform team - can build a foundation for shared resources that includes built-in reasonable boundaries to enforce constraints to prevent issues with costs and security.

The goal of this platform team should be to empower developers to autonomously build their applications in the Cloud. This platform team does not need to be directly in the middle of their development lifecycle. Through automation, security risks are either removed or brought to attention. And it gets better: these risks can be mitigated before they even hit the environment.

I have had the privilege to build such platforms in Amazon Web Services (AWS) in the last few years. In this blog post I'll go through some of the tools and processes that are available to grant this autonomy.

## 1. Preventing public S3 buckets

There are many examples of (big) companies leaking (sensitive) data through public S3 buckets. Many companies have policies these days to disallow the use of public buckets. If you do want to make data (publicly) available, you should either put [CloudFront](https://aws.amazon.com/cloudfront/) in front of it or use [Presigned Urls](https://docs.aws.amazon.com/AmazonS3/latest/dev/PresignedUrlUploadObject.html).

Since last year it is now possible to [fully block public access](https://docs.aws.amazon.com/AmazonS3/latest/dev/access-control-block-public-access.html) to all S3 buckets (or specific ones). This is a configuration separate from the existing bucket policies and ACLs. Even if your policy or ACL allows public access, this new setting will still block it.

Check out the referenced documentation to learn how to switch on this setting, and be sure that none of your buckets are open to the world. It's currently not yet possible to enable this configuration through CloudFormation. I wrote a blog post earlier that allows you to do this [with a custom resource](https://sanderknape.com/2018/11/blocking-account-wide-creation-public-s3-buckets-cloudformation-custom-resource/).

## 2. AWS Config

[AWS Config](https://aws.amazon.com/config/) is the de facto tool within AWS to enforce your company policy onto all AWS resources. It allows you setup certain rules for specific resources and have AWS Config take action for any resources that don't follow these rules. An action can be to send a notification for someone to act, or AWS Config may remove the resource immediately.

Many managed rules [already exist](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config_use-managed-rules.html) that you can start using directly. You can also add your own custom rules by having a Lambda function invoked.

In general there are two ways to deal with resources that violate policies:

1. _Remove or modify the resource_. The easiest way to deal with violating resources is by simply removing them. Or perhaps it's enough to simply change the specific setting that violates your company policy (e.g. remove public access from an S3 bucket or tighten an open SSH port on a security group). If a resource may be an immediate security risk I would definitely recommended doing this. However, this action may not be very "user-friendly" to the developer who has created this resource. Instead of allowing the developer to fix his own mistake, you have forcefully "fixed" it for him. His CloudFormation or Terraform state will be out of sync which may be hard to straighten. If possible, give the developer a period of time in which he can fix the violating resource himself.
2. _Notify and provide insights_. If you choose not to remove or modify a resource that violates a policy, you should at least make sure that the right person is notified about the violation. This may for example be a security team or a platform team and ideally the team that introduced the violation. At the same time there should be a clear overview of current resources that are violating a policy and for how long they have been in violation.

In addition, you can use AWS Config to enforce the use of certain tags. This is typically required to get insights into the resources different teams/applications are using. This way, you can get insights into the costs per team/application. Check out this AWS blog to learn how to [enforce tags with AWS Config](https://aws.amazon.com/blogs/devops/aws-config-checking-for-compliance-with-new-managed-rule-options/).

### Using other tools

AWS Config is just one of the tools that can enforce and provide insights on policies in AWS. Tools like [CloudHealth](https://www.cloudhealthtech.com/) and [CloudCheckr](https://cloudcheckr.com/) provide similar functionalities. In addition, these tools support additional features such as generating cost insights and providing insight into multiple Cloud providers. Depending on your use case they may be more relevant for you compared with AWS Config.

One tool I've personally enjoyed using is [CloudSploit](https://cloudsploit.com/). It's a simple, no-nonsense tool that scans your AWS environment for security and policy violations. The scans it performs are open source so you can see exactly what it checks in your environment. Compared to the previous tools it doesn't provider additional features such as cost insight. This is honestly what I like, as it does one thing specifically and it does it really well.

I would personally recommend running an external tool in addition to AWS Config. These tools have more built-in scans than AWS Config and you therefore have to write less custom checks yourself. I also appreciate the idea of having an external entity scan my environment, instead of me writing my own rules that are potentially flawed.

## 3. Multiple accounts and shared networking

In some cases you not only want to prevent data being available to the big bad outer world, but to other applications or entities within the same organization. You may have stored passwords in [Secrets Manager](https://aws.amazon.com/secrets-manager/), or sensitive data in [DynamoDB](https://aws.amazon.com/dynamodb/). And only the team that has build this should be allowed to access this.

It's possible to set this up when there are multiple (whether it's a few, dozens or hundreds) of teams in a single AWS account. You can ensure that the different teams get different IAM roles assigned. These will allow you for example to manage and see only secrets/databases that are prefixed with your team name. However, it some point it can just become annoying. With dozens or hundreds of teams you will reach limits within the AWS account. When you login to the console, your resources are mixed with resources from many different other teams.

Instead, you can make multiple accounts, for example one per application, one per team or some other logical way depending on your organization. Just be careful not to mirror it too much with the organization. Before you know it there is a (either big or small) restructuring of the team setup, and your AWS setup is now out of date. Aligning this may take a lot of effort.

For a long time a challenge with multiple accounts was network connectivity. As each account has its own VPC, you would need to set up VPC peering to each VPC. With dozens of accounts this would add a lot of complexity. Instead, you can now use [VPC Sharing](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-sharing.html). In a single account (perhaps owned by a central networking/platform team) you create a VPC, and share this VPC with the other accounts in which developers deploy their applications. You now have the benefit of multiple, isolated accounts, as well as easy networking connectivity between the various applications.

Another option is to use [Transit Gateway](https://aws.amazon.com/transit-gateway/). If you do have multiple VPCs in different accounts between which you want to enable private networking, you can connect all these VPCs to this single gateway. From a central location you can then easily configure the exact routing rules that you need, and only access between the VPCs that need this. Through the same mechanisms as described in this blog post you can then enable the development teams to manage this configuration themselves.

To learn more about these relatively new networking features, check out [this video](https://www.youtube.com/watch?v=fnxXNZdf6ew) from re:Invent last year.

## 4. IAM Permissions Boundary

Using [AWS IAM](https://aws.amazon.com/iam/) you lock down your environment by only allowing your deployment pipelines to do certain things. If you don't want to have any virtual machines deployed in certain regions, you simply prevent them by not allowing it in the IAM policy attached to the deployment pipeline.

However, these deployment pipelines can probably create additional IAM resources. You may want to deploy an EC2 instance with an IAM role attached. What prevents you from giving that IAM role admin permissions, and then being able to do anything you want from that instance?

That's where [permissions boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) come in. A permissions boundary is just another IAM policy that is attached to an IAM entity (a role, user or group). The permissons boundary defines the _maximum permissions_ which that entity will have. If the entity has fewer permissions configured, it only has those permissions. If it configures more permissions than allowed within the boundary, the permissions are capped to whatever is configured in the boundary.

To enforce that any IAM resource created through your deployment pipelines have the permissions boundary attached, you can use the `Condition` statement within the IAM policy to do so (this example is copied from the documentation referenced above):

```json
{
    "Sid": "CreateOrChangeOnlyWithBoundary",
    "Effect": "Allow",
    "Action": [
        "iam:CreateUser",
        "iam:DeleteUserPolicy",
        "iam:AttachUserPolicy",
        "iam:DetachUserPolicy",
        "iam:PutUserPermissionsBoundary"
    ],
    "Resource": "*",
    "Condition": {
        "StringEquals": {
            "iam:PermissionsBoundary": "arn:aws:iam::111122223333:policy/XCompanyBoundaries"
        }
    }
},
```

To learn more about permissions boundaries I would definitely recommend watching [this video](https://www.youtube.com/watch?v=eVNvjQ0wr84) from re:Inforce. It's a fun watch and shows a hands-on example of how to setup permissions boundaries.

## 5. Policies in the pipeline

These days it's very common to write Infrastructure as Code (IaC) to provision (cloud) environments. One of the advantages of using IaC is that you have specified your environment in a formalized, structured way. What this means is that you can run scans on these specifications _before_ they are deployed to the environment. Instead of _reacting_ to configuration changes with a tool like AWS Config, you _proactively_ prevent introducing policy violations by catching them before they are deployed.

Looking at your environment from a platform perspective, the sum of the boundaries that you set in AWS essentially define a "contract" to your developers on how they may use the platform. By scanning the IaC as part of their pipelines you have formalized that contract and prevent them for violating it before it gets in.

In addition, this mechanism provides early feedback. It's much more convenient to learn you have to change something as feedback within a pull request. Or even better: directly in your editor.

If you use CloudFormation for IaC, definitely have a look at the [CloudFormation Linter](https://github.com/aws-cloudformation/cfn-python-lint). It can lint your CloudFormation for errors before deploying your stack. This by itself is already great as you will deploy erroneous CloudFormation templates much less with using this.

What is even better is that you can extend the linter with your own company policies. This, combined with the [editor plugins](https://github.com/aws-cloudformation/cfn-python-lint#editor-plugins) means that developers can already get feedback on violations the moment they write it in their editor. As soon as they create a security group rule that opens up SSH to the world, that line will get a red underline. In terms of "shift left" and "early feedback", this is as left and early as you can get it.

If you are using Terraform Enterprise you can use [Sentinel](https://www.terraform.io/docs/enterprise/sentinel/import/index.html). If you are using the free version you can take a look at [conftest](https://github.com/instrumenta/conftest). I haven't yet seen a way to get feedback directly in the editor, but at least you can run a scan in the pull request. For example, the following will scan a Terraform plan for the existence of permissions boundaries:

```
package main

permissions_boundary_arn = "arn:aws:iam::0123456789:policy/permissions-boundary-policy"

iam_resources = [
  "aws_iam_user",
  "aws_iam_role",
  "aws_iam_group"
]

deny[msg] {
  input.planned_values.root_module.resources[i].type = iam_resources[j]
  not input.planned_values.root_module.resources[i].values.permissions_boundary = permissions_boundary_arn

  msg = sprintf("IAM resource '%s' must have 'permissions_boundary' property set to '%s'", [input.planned_values.root_module.resources[i].address, permissions_boundary_arn])
}
```

## Conclusion

The combination of the tools above allow you to provide developer autonomy in AWS. In addition, it does so in a pretty developer-friendly way. Especially when providing early feedback you can help your developers quickly learn they are configuring something that falls outside of the company policies. Make sure you provide documentation that explains exactly why these company policy are setup. I believe that providing this autonomy to developers should be the main goal for any platform team. The development teams can build their applications faster, and the platform team has more focus to work on larger projects instead of constantly being distracted by developers who need their help to get something done.

Again: it _is_ possible for a platform team to remove yourself from the development's software lifecycle. Instead, you will actually start communicating about and collaborating on the things that really matter.

Did I miss anything? Are there any other risks that I haven't covered in this post? If so, let me know!
