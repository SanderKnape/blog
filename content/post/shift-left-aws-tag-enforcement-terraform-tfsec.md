+++
title = "Shift left AWS tag enforcement with Terraform and tfsec"
author = "Sander Knape"
date = 2021-05-03T21:03:01+02:00
draft = false
tags = ["terraform", "shift left", "ci/cd", "tfsec", "security", "aws"]
categories = []
+++
There are many ways to improve the developer experience of deploying infrastructure into the Cloud. One such method is by shifting left: provide early feedback to shorten the feedback loop and speed up development.

When deploying infrastructure into AWS with an infrastructure as code tool such as Terraform, you can validate that code as part of a CI/CD pipeline. A pull request can automatically receive feedback about the configuration of resources, thus enforcing the environment to stay compliant with the organization's policies.

One enforcement you may want to implement is tagging. Structural and consistent tagging allows you to quickly see to which application, team or cost centre a resource belongs. Such tags make it easier to debug issues, get cost insights or to find out who to contact about a resource.

In this blog post I'll share how a recent release of the [Terraform AWS Provider](https://github.com/hashicorp/terraform-provider-aws) makes enforcing tags more effortless than ever before.

## Terraform AWS Provider v3.38.0

Last week, version [3.38.0](https://github.com/hashicorp/terraform-provider-aws/releases/tag/v3.38.0) of the Terraform AWS Provider made the [default tags](https://registry.terraform.io/providers/hashicorp/aws/latest/docs#default_tags) feature generally available. It is now possible to set tags on the AWS provider configuration and automatically tag all resources that support tags.

This is extremely convenient. Before, you'd have to maintain a list of resources that support tags and enforce the existence of tags for all those specific resources. Now, you only have to check for the existence of the `default_tags` on the AWS provider configuration.

The following is an example configuration of setting default tags on the provider:

```hcl
provider "aws" {
  default_tags {
    tags = {
      Application = "Bazinga"
    }
  }
}

resource "aws_sqs_queue" "queue" {
  tags = {
    Foo = "Bar"
  }
}
```

When planning this configuration, you will see that Terraform will deploy the queue with both the `Application` tag and the `Foo` tag.

```
...

Terraform will perform the following actions:

  # aws_sqs_queue.queue will be created
  + resource "aws_sqs_queue" "queue" {
      + arn                               = (known after apply)
      ...
      + tags                              = {
          + "Foo" = "Bar"
        }
      + tags_all                          = {
          + "Application" = "Hello"
          + "Foo"         = "Bar"
        }
      + visibility_timeout_seconds        = 30
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

When you set the same tag key both in the provider as in a resource, the value specified in the resource will take precedence.

## tfsec
One tool that can analyze your Terraform code is [tfsec](https://github.com/tfsec/tfsec). By default, it includes many checks for AWS, Azure, GCP as well as some general checks. Especially useful is how easy it is to add [custom checks](https://tfsec.dev/docs/custom_checks/). You can add whatever check you want to enforce that all deployed infrastructure is compliant and follow the best practices defined by your organization.

You can also run `tfsec` locally, giving developers the option to get even earlier feedback on their Terraform manifests. Keep that developer experience in mind.

The following is a custom `tfsec` check that checks for the existence of the `Application` tag on the provider level:

```yaml
checks:
  - code: CUS001
    description: Check if the AWS provider has the default "Application" tag set for all supported resources.
    requiredTypes:
      - provider
    requiredLabels:
      - aws
    severity: ERROR
    matchSpec:
      name: default_tags
      action: isPresent
      subMatch:
        name: tags
        action: contains
        value: Application
    errorMessage: The Application tag in the "default_tags" configuration of the AWS provider was missing.
    relatedLinks:
      - https://registry.terraform.io/providers/hashicorp/aws/latest/docs#default_tags
```

With the [tfsec GitHub Action](https://github.com/triat/terraform-security-scan) it is easy to add an automated check to your pull request. The following GitHub workflow will analyze your Terraform.

```yaml
name: tfsec

on: pull_request

jobs:
  tfsec:
    name: tfsec
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
      - uses: triat/terraform-security-scan@v2.0.2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

This is what the feedback will look like as a comment in a pull request for a found issue ([source](https://github.com/SanderKnape/tfsec-enforce-tagging/pull/1)).

![](/images/tfsec-feedback.png)

I've created a fully working [sample repository](https://github.com/SanderKnape/tfsec-enforce-tagging) that showcases the functionality end-to-end, including the above custom check. Give it a try and see how you can improve how you deploy infrastructure into the Cloud.
