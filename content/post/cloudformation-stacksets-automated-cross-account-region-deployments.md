+++
title = "CloudFormation StackSets: automated cross-account/region deployments"
author = "Sander Knape"
date = 2017-07-26T19:53:02+02:00
draft = false
tags = ["automation", "aws", "cloudformation"]
categories = []
+++
Yesterday, AWS released [CloudFormation StackSets](https://aws.amazon.com/blogs/aws/use-cloudformation-stacksets-to-provision-resources-across-multiple-aws-accounts-and-regions/). A StackSet is a set of CloudFormation stacks that can easily be deployed to multiple AWS accounts and/or multiple AWS regions. Before, each stack had to be deployed separately and custom scripts were required to orchestrate deploying to multiple accounts/regions. Therefore, this feature is bound to make the lives of AWS administrators a bit easier.  

There are loads of use cases for deploying stacks to multiple locations. For example, it's considered a best practice to [enable AWS Config in every region](https://www.slideshare.net/AmazonWebServices/aws-security-best-practices). This service keeps track of resources in an AWS account and changes to those resources. AWS Config needs to be enabled in every region separately, so a CloudFormation stack is required for every region.  

Another use case is sandbox account. If you have a set of sandbox accounts for software engineers in your company, you want to keep these accounts in the same state. Instead of provision CloudFormation stacks in every account separately, you can now use a single StackSet to provision all accounts with a single API call.  

The feature announcement from AWS already includes instructions on how to set this up through the AWS Console. Us AWS pros of course want to provision our accounts automatically, so let's see how we can use the newly added AWS CLI methods to provision a StackSet in multiple regions.

# Getting started

We will use the AWS CLI to create a StackSet with a very simple CloudFormation stack. First, make sure you install the latest version; at the time of writing this feature has been added to the CLI a mere 20 hours ago. As described by AWS, the easiest method to install the AWS CLI is through [pip](http://www.pip-installer.org/en/latest/):

```bash
pip install awscli

aws --version # should be at least 1.11.125
```

Next, [correctly set up your credentials](https://github.com/aws/aws-cli#getting-started). For this blog post, I'm using an (not so best practice) IAM role with the Admin policy. All we will do is create a CodeDeploy application, so feel free to use a role with more fine-grained permissions.  

Before we're going to provision a CloudFormation stack, let's dive a bit into the concept of StackSets first. Though it's not visible from the AWS Console, in the background AWS actually manages two different types of resources:

1.  The StackSet. A StackSet is the container in which the set of CloudFormation stack instances will live. When creating a StackSet, we pass it the CloudFormation YAML/JSON. In addition, we can optionally pass it a set of parameters and tags for the CloudFormation stack.
2.  A Stack resource. A Stack resource is an actual instantation of the template we provided to the StackSet. This Stack lives in a specific AWS account and in a specific region. If we provision a CloudFormation template to three AWS accounts and in five different regions, we have a single StackSet but fifteen Stack resources.

You might notice a limitation in my description: parameters are passed to the _StackSet_ instead of a Stack resource. This means that we can only provide a single set of parameters that will be used for _every_ Stack resource. In other words: we can't define different parameters for different resources. This would unlock a bunch of more use cases, so let's hope this is something that will be added in the future. (_EDIT November 18th, 2017: it is now possible to [provide different parameters per stack](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stackinstances-override.html)_).

## IAM permissions

CloudFormation needs some very specific permissions to get a StackSet up and running. First, we need to create an IAM role called `AWSCloudFormationStackSetAdministrationRole` in what is called the "administrator account". This is the account from which we create the StackSet and from where we'll deploy the stacks in other accounts and regions. Next, we have the `AWSCloudFormationStackSetExecutionRole` IAM role. This role must exist in all accounts where we're going to provision stacks. The first role will assume the latter role. For specific examples on how to create these roles, check out the [AWS documentation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs.html).  

Everything combined, the overview looks as follows:  

![CloudFormation StackSets IAM setup.](/images/stacksets_iam.png)  

In this blog post, we're going to keep it a little simpler. We'll create both IAM roles in the same account and we'll deploy the Stack instances only in a single AWS account (though, of course, in multiple regions). In other words: your AWS account is both the administrator account and the target account. I can imagine not everyone has multiple accounts lying around!

# Provisioning our StackSet and Stack resources

With your IAM credentials and roles setup, we're going to deploy the simplest CloudFormation that I can think of (I challenge you to show me an easier one!):

```yaml
Resources:
  MyApplication:
    Type: "AWS::CodeDeploy::Application"
```

Save this as `stack.yaml`. All it does is create a CodeDeploy application without any deployment groups, so it won't actually do anything (nor cost anything for that matter). Next, execute the following API call to create the StackSet:  

`aws cloudformation create-stack-set --stack-set-name my-codedeploy-application --template-body file://stack.yaml`  

This will create the StackSet and you'll see the StackSet ID being returned (we don't actually need this). Looking in the AWS Console, we see the StackSet is properly created:  

![The CloudFormation StackSet is created.](/images/stackset.png)  

If you click on the StackSet, you'll notice there aren't any Stacks instantiated yet. We can however view the template and any parameters/tags that we could have specified.  

Next, it's time to get some stacks up and running. With the following command, we'll provision the template in three different regions:  

`aws cloudformation create-stack-instances --stack-set-name my-codedeploy-application --accounts YOUR_ACCOUNT_ID --regions "eu-west-1" "us-east-1" "us-east-2"`  

Replace the YOUR\_ACCOUNT\_ID part with your AWS account ID. This is the account where we'll instantiate all the stacks. We've specified three different regions, but we could have even chosen to provision the stack in all regions. Unfortunately, as far as I can tell, there isn't an option to simply specify "ALL" regions other than specifically listing all regions. This would be a nice addition, for example in the event where a new region is added. Stacks that must be deployed in all regions will then automatically be provisioned in the new region as well. And, of course, "ALL" is much more clear than a list of every region.  

Anyway, going back to the AWS Console we see that the stack has properly been deployed in every region:  

![The CloudFormation Stack instances are deployed in three regions.](/images/stack_instances_deployed.png)  

Next, let's give our CloudFormation template a small update. Change it so it looks like this:

```
Resources:
  MyApplication:
    Type: "AWS::CodeDeploy::Application"
  MyApplication2:
    Type: "AWS::CodeDeploy::Application"
```

Exciting: we're not deploying one, but two empty CodeDeploy applications! Save the file and run the following API call to update the StackSet:  

`aws cloudformation update-stack-set --stack-set-name my-codedeploy-application --template-body file://test.yaml`  

Again, take a peek at the CloudFormation Console and you'll see the regions are updated one by one.

# Cleaning up

Finally, it's time to clean up. If we just try to delete the entire StackSet, we get the following error:  

`aws cloudformation delete-stack-set --stack-set-name my-codedeploy-application An error occurred (StackSetNotEmptyException) when calling the DeleteStackSet operation: StackSet is not empty`  

Ah: looks like we need to delete all the Stack instances within the StackSet first. This makes sense, considering that the StackSet is simply a container that orchestrates the provisioning and updates of stacks across accounts and/or regions. Therefore, let's remove all the Stack instances first:  

`aws cloudformation delete-stack-instances --stack-set-name my-codedeploy-application --accounts YOUR_ACCOUNT_ID --regions "eu-west-1" "us-east-1" "us-east-2" --no-retain-stacks`  

The final option `no-retain-stacks` will actually delete the stacks. The reverse option, `retain-stacks`, will only disassociate the stacks from the StackSet but not actually remove the stacks. With the Stack instances removed, run the above `delete-stack-set` command again to remove the StackSet and you'll see its gone.

# Conclusion

CloudFormation StackSets is certainly a welcome new feature that will make the lives for AWS administrators easier. Of course, this is just the first version of this great new functionality. I'm hoping the following limitations are already somewhere on the roadmap to be addressed:

*   Only a single set of parameters can be defined. Consider the use case of Sandbox environments that all have non-overlapping VPCs CIDR blocks: we'd like to provide different blocks per account. Currently, that's not possible by specifying different parameters to our Stack instances. Dozens of other use cases exist where this would make sense. (_EDIT November 18th, 2017: it is now possible to [provide different parameters per stack](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stackinstances-override.html)_).
*   There is no way to specify "ALL" AWS regions. This would be great for both readability and maintainability.
*   It's possible to specify the order in which stacks are deployed to accounts/regions. This allows you to provision a stack to a staging environment first before provisioning it to production (you can configure it to stop on failure). However, even better would be if we could run an automated test in-between those two deployments. Of course, this is something that is possible in AWS CodePipeline but with that tool, [some hacking is required](https://aws.amazon.com/blogs/devops/building-a-cross-regioncross-account-code-deployment-solution-on-aws/) to set it up multi-account/multi-region. In other words: if AWS would somehow bring these two tools closer together, we would get the best of both worlds.
