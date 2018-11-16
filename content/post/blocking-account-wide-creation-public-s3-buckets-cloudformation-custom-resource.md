+++
title = "Blocking account-wide creation of public S3 buckets through a CloudFormation custom resource"
author = "Sander Knape"
date = 2018-11-16T17:25:02+02:00
draft = false
tags = ["s3", "cloudformation", "lambda"]
categories = []
+++
Yesterday, AWS announced the release of an important and much-wanted new feature for S3: [blocking the creation of public S3 buckets on an account-wide](https://aws.amazon.com/blogs/aws/amazon-s3-block-public-access-another-layer-of-protection-for-your-accounts-and-buckets/). Enough has been written already about open S3 buckets on the internet. Given that it is very simple to create a public S3 bucket, we regularly learn about new (big) companies that have exposed privacy-sensitive data to the world through such buckets.

The confusion is mainly around opening up your bucket to "everyone". Where people expect this to mean "everyone in the AWS account", it actually means "*everyone in the world*".

![ACL for a public S3 bucket.](/images/public_s3_bucket_acl.png)  

Even though a big warning is visible, it doesn't tell you explicitly that "everyone" means "everyone in the world".

Similarly with S3 bucket policies, granting "everyone" access to your bucket again means the entire world.

![Policy for a public S3 bucket.](/images/public_s3_bucket_policy.png)  

While I’ll ignore the S3 ACL vs. bucket policy discussion, it suffices to say that there are many ways to open up your bucket to the world. Luckily, last year AWS made it much more visible that you have public buckets through a simple UI update.

![Policy for a public S3 bucket.](/images/public_s3_bucket_ui.png)  

While such UI changes certainly help, they are still reactive. While they tell you which buckets are public, they don't block you in creating any new buckets.

I was very happy to read in the announcement that CloudFormation support has been added already for the S3 Bucket resource. However, I didn't see any information regarding account-wide permissions for blocking the creation of public S3 buckets. In this blog post, I'll share how this can be done using the CLI and I will share a CloudFormation custom resource that you can use to block the creation of S3 buckets in code.

# The new S3 API for account-wide access control

As part of the release yesterday, AWS updated their API with new account-wide permissions for S3 through the [s3control API](https://docs.aws.amazon.com/cli/latest/reference/s3control/index.html#cli-aws-s3control). This API contains three commands for manipulating the account-wide permissions:

* delete-public-access-block
* get-public-access-block
* put-public-access-block

Given an AWS account where the account-wide access controls have not been changed yet, the `get-public-access-block` command will return these values:

```bash
aws s3control get-public-access-block --account-id [youraccountid]
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": false,
        "IgnorePublicAcls": false,
        "BlockPublicPolicy": false,
        "RestrictPublicBuckets": false
    }
}
```

Note: if you get an error that this API call does not exist, be sure to update the AWS CLI to the latest version.

Please check out the [original blog post](https://aws.amazon.com/blogs/aws/amazon-s3-block-public-access-another-layer-of-protection-for-your-accounts-and-buckets/) for more information regarding these four settings. Through the following command, you can block the creation of public S3 buckets in the future. 

> ! **Important**: this command will also update *existing* buckets. If you currently rely on the use of public S3 buckets, this command will **BREAK** that functionality. Do **NOT** run this command if you are intentionally using public S3 buckets.

```bash
aws s3control put-public-access-block --account-id [youraccountid] --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPol
icy=true,RestrictPublicBuckets=true"
```

This feature is also available through the [boto3 S3Control API](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3control.html), which means that we can create a CloudFormation custom resource that blocks the creation of any new public S3 buckets in your entire account.

# A CloudFormation custom resource for blocking public S3 buckets

I’m a strong advocate for “everything in code”. Therefore ideally, we store this new configuration in desired state. For me, CloudFormation is the way to go in AWS. You can find the full source code in my [GitHub repository](https://github.com/SanderKnape/block-s3-buckets-cloudformation-custom-resource).

Using the [cfn-lambda-handler](https://github.com/mixja/cfn-lambda-handler) decorators, it’s very easy to create a custom resource with just a few lines of code. The full source code for the Lambda function is as follows:

```python
import os
import boto3
from cfn_lambda_handler import Handler

client = boto3.client('s3control')
handler = Handler()

@handler.create
@handler.update
def handle(event, context):
    client.put_public_access_block(
        AccountId=os.environ['ACCOUNT_ID'],
        PublicAccessBlockConfiguration={
            'BlockPublicAcls': True,
            'IgnorePublicAcls': True,
            'BlockPublicPolicy': True,
            'RestrictPublicBuckets': True
        }
    )

    return { "Status": "SUCCESS" }

@handler.delete
def handle_delete(event, context):
    client.delete_public_access_block(
        AccountId=os.environ['ACCOUNT_ID']
    )

    return { "Status": "SUCCESS" }
```

The AWS account ID is required for these API calls. We therefore inject the account ID through an environment variable from the CloudFormation resource. The custom resource simply wraps the two API calls in two different handlers.

The following [Serverless Application Model](https://github.com/awslabs/serverless-application-model) deploys this custom resource:

```yaml
Transform: AWS::Serverless-2016-10-31

Resources:
  BlockPublicS3BucketsFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: build/
      Handler: publicbuckets.handler
      Runtime: python3.6
      Timeout: 5
      Environment:
        Variables:
          ACCOUNT_ID: !Ref AWS::AccountId
      Policies:
        - Version: 2012-10-17
          Statement:
            - Effect: Allow
              Action:
                - s3:* # See comment below
              Resource: "*"

Outputs:
  BlockPublicS3BucketsFunction:
    Value: !GetAtt "BlockPublicS3BucketsFunction.Arn"
    Export:
      Name: "custom-resource-block-public-s3-buckets"
```
> At the time of writing, I have not yet been able to find the specific IAM permissions needed to execute the `put_public_access_block` and the `delete_public_access_block` API calls. I have of course tried the obvious (`s3:DeletePublicAccessBlock` and `s3:PutPublicAccessBlock`), but these do not work. As it works with `s3:*`, I have decided to use this for now. As this is definitely not following the security least-privilege principle, I would very much like to replace this with the proper permissions. If you happen to know what permissions to place here, please leave a comment below or send me a message on [Twitter](https://twitter.com/SanderKnape).

The following stack shows his this resource can now be used:

```yaml
Resources:
  BlockPublicS3Buckets:
    Type: "Custom::BlockPublicS3Buckets"
    Properties:
      ServiceToken: !ImportValue "custom-resource-block-public-s3-buckets"
```

I would advice using this custom resource in a global foundation/infrastructure/account stack that you may have defined. You just need it once to block any current or future public S3 buckets in your account.

Again, the entire deployable stack is available in my [GitHub repository](https://github.com/SanderKnape/block-s3-buckets-cloudformation-custom-resource), including deployment instructions.

# Conclusion
I was very happy to read that it is now possible to [block the creation of public S3 through an account-wide setting](https://aws.amazon.com/blogs/aws/amazon-s3-block-public-access-another-layer-of-protection-for-your-accounts-and-buckets). Unfortunately, the blog post only contains examples for specifying these permissions on a single bucket. Also, I found that it is not possible yet to block public s3 buckets account-wide through CloudFormation.

Therefore, in this blog post I explain how you can use the AWS CLI to block the creation of public s3 buckets on an account-wide level. In addition I have created a CloudFormation custom resource to block these permission in a desired-state configuration. I hope the custom resource is of use to anyone!
