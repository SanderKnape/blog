+++
title = "A custom CloudFormation resource example for CodeDeploy"
author = "Sander Knape"
date = 2017-08-17T21:25:02+02:00
draft = false
tags = ["automation", "aws", "cloudformation", "codedeploy", "lambda"]
categories = []
+++
CloudFormation is the AWS product for Infrastructure as Code. It allows you to provision AWS resources through a template that describes how to configure that resource. Unfortunately, CloudFormation will sometimes be behind on new features released by AWS. Where the AWS console and API will allow you to deploy resources with a certain configuration, in CloudFormation specific settings might simply not yet be available. If its your goal to deploy your AWS environment completely through Infrastructure as Code, this will block you from doing that. Luckily, through [custom resources](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html), CloudFormation allows you to extend the templating language and still give you the option to provision resources entirely through CloudFormation.

CloudFormation custom resources are basically wrappers that point towards a Lambda function. The custom resource references a Lambda function that receives a set of properties you define in the CloudFormation template. The Lambda function can then do whatever you want, though most common is the use an AWS SDK to make changes to your resources that are not yet available in CloudFormation.

Recently I was looking for some concrete CloudFormation custom resource examples and I noticed these are rather scarce. Therefore, in this blog post, I present a concrete example for a CloudFormation custom resource. We are going to add some extra functionality for CodeDeploy. Let's get started!

## CodeDeploy

At the time of writing, [a recently released feature for CodeDeploy](https://aws.amazon.com/about-aws/whats-new/2017/05/aws-codedeploy-now-integrates-with-elastic-load-balancing/) is not yet available through CloudFormation. With this feature, you can specify a classic load balancer from which CodeDeploy will remove an EC2 instance during the deployment. First, the load balancer will not accept new traffic to that instance. Then, after a specified amount of time, the instance will be completely removed from the load balancer. Finally, the instance is updated after which CodeDeploy will put the instance back in the load balancer.

Even more recently (last week), AWS announced [support](https://aws.amazon.com/about-aws/whats-new/2017/08/aws-codedeploy-now-supports-using-application-load-balancers-for-blue-green-and-in-place-deployments/) for the application load balancer. The concepts explained in this blog post will work for the application load balancer as well. Instead of a load balancer, you specify a target group to which to connect CodeDeploy to. Or, you can extend the function to allow either a classic load balancer reference or a target group, and setup the CodeDeploy configuration based on which property the user defined in CloudFormation.

The [CloudFormation DeploymentGroup resource](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-codedeploy-deploymentgroup.html) does not support a property for specifying this load balancer (again: at the time of writing! Let's hope this is added soon). With the AWS Javascript SDK, this is [already possible](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/CodeDeploy.html). Therefore, we can create a Lambda function that handles this logic for us.

We are going to spin up a CloudFormation stack that contains all the resources necessary to setup a CodeDeploy deployment group. It will also create the Lambda function that is executed through the custom resource and associate a load balancer with CodeDeploy using this custom resource.

## The CloudFormation Stack

Check out the entire CloudFormation stack in [my GitHub repository](https://github.com/SanderKnape/cloudformation-custom-resource-example). This stack contains all the resources required to setup CodeDeploy, a load balancer and the custom resource. Also, the Lambda function is in there that is executed through the custom resources.

Let's go through all resources in a little more detail;

*   **CodeDeployApplication**, **CodeDeployDeploymentGroup** and **CodeDeployIAMRole**. These together setup CodeDeploy. Because we're not actually going to deploy anything, the IAM Role does not have a policy attached and thus no permissions.
*   **EC2Instance**. It's not possible to deploy a CodeDeploy DeploymentGroup without specifying an instance to deploy to, even if we're not going to deploy anything. Therefore we'll spin up an instance.
*   **ElasticLoadBalancingLoadBalancer**.This is the classic load balancer that we associate with the CodeDeploy deployment group. You can also specify a target group instead of the classic load balancer. The AMI ID I use here is valid for the **eu-west-1** region. As AMI IDs are different across regions, you'll need to change it if you want to deploy this stack in a different region.
*   **CodeDeployElbAssociation**.The Lambda function is executed through the custom resource, represented as the CodeDeploy ELB Association (we're associating a load balancer with CodeDeploy... hence the name). With the ServiceToken we specify the Arn of the Lambda function that must be executed. In this case, the Lambda function is in the same stack. If you use the same custom resource in multiple stacks, you must hardcode the Arn or better: use an [exported value](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html).
*   **LambdaFunction** and **LambdaIAMRole**. We give Lambda full CodeDeploy and CloudWatch (for logging debug info) permissions: keep in mind that for your use case, you can probably make this more fine grained (for example, the Lambda function doesn't need to delete a deployment group). I explain the Lambda function itself in more detail in the next section.

My goal was to keep the stack as minimal as possible, so I'm using all default configuration as far as possible or I chose the simplest configuration. Sometimes, this doesn't make sense such as spawning the load balancer outside of a VPC (because I specify availability zones instead of subnets). For our use case this doesn't matter though, but keep it in mind and please don't think I'm an idiot :-).

Deploy this stack through the CloudFormation interface. After a few minutes, it will be done and has successfully associated the ELB with the CodeDeploy deployment group. If you log into the AWS Console and open up the CodeDeploy deployment group, you will see the following:

![Shows a CodeDeploy deployment group configuration with an associated classic load balancer](/images/codedeploy_with_elb_association.png)

Let's go into a bit more detail to see how the custom resource works.

## Invoking the Lambda function

After provisioning the stack, check out the CloudWatch logs for the Lambda function. It outputs the event that is received from CloudFormation. This looks something like this:

<pre>
{
  <strong>RequestType</strong>: 'Create',
  ServiceToken: 'arn:aws:lambda:eu-west-1:xxx:function:codedeploy-elb-association',
  <strong>ResponseURL</strong>: 'https://cloudformation-custom-resource-response-euwest1.s3-eu-west-1.amazonaws.com/temporary-sts-token-credentials',
  StackId: 'arn:aws:cloudformation:eu-west-1:xxx:stack/codedeploy/5cde41b0-7e07-11e7-b4e6-500c3d3cda29',
  RequestId: 'cc3b2f39-2b35-4751-8c2b-672b126b56d3',
  LogicalResourceId: 'CodeDeployElbAssociation',
  ResourceType: 'Custom::CodeDeployElbAssociation',
  <strong>ResourceProperties</strong>: {
    ServiceToken: 'arn:aws:lambda:eu-west-1:xxx:function:codedeploy-elb-association',
    LoadBalancerName: 'codedeplo-ElasticL-1SBDCSZ6HSC76',
    CodeDeployDeploymentGroupName: 'my-codedeploy-deployment-group'
  }
}
</pre>

The interesting bits are bolded. First, we receive a **ResponseURL** to which we need to respond with either a success or failure message. This is abstracted away for us through the `cfn-response` library. Second, we receive the **ResourceProperties** specified in the CloudFormation template. As you can see, the references are properly replaced with the names of the load balancer and deployment group. This means that our Lambda function has enough information to associate the two through the use of the Javascript SDK.

Finally, we receive a **RequestType** of "Create" in this example. There are three valid options here: Create, Update and Delete. Typically, Create and Update can be implemented the same whereas Delete will remove the custom resource. In our use case, that means removing the association between the load balancer and CodeDeploy. We do **not** remove either the load balancer or the CodeDeploy deployment group! These are managed through the resources defined in the CloudFormation stack.

Looking at the Lambda function, you will notice that indeed for Delete we specify the "WITHOUT\_TRAFFIC\_CONTROL" setting. Otherwise, we specify "WITH\_TRAFFIC\_CONTROL" and specify the load balancer that we received through the `ResourceProperties`. Finally, we use the [updateDeploymentGroup](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/CodeDeploy.html#updateDeploymentGroup-property) command from the SDK to either associate or disassociate the load balancer from the deployment group.

Keep in mind that you only have to deploy the Lambda function once. If you need to use it in multiple stacks, use an exported value to share the Arn with different stacks.

## Conclusion

It's mainly the lack of concrete and clear custom CloudFormation resource examples that made me write this blog post. In essence, it's very simple: you write a Lambda function that receives and event from a custom resource. Through the properties, we send it any info we need. We can safely use [CloudFormation intrinsic functions](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html) to specify names, ARNs or anything else we need to dynamically query resource properties that are only known after the resources are created. Then, in the Lambda function, we implement the functionality that is already available through the SDK, but not yet through CloudFormation.

So, if its your goal to put your entire AWS infrastructure in CloudFormation templates, definitely look into custom resources if you run into missing features! It's very easy, so give it a shot.
