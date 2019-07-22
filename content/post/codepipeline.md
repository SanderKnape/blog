+++
title = "CodePipeline and CodeBuild"
author = "Sander Knape"
date = 2019-04-19T20:39:02+02:00
draft = false
tags = ["aws", "codepipeline", "codebuild", "ci/cd"]
categories = []
+++
[AWS CodePipeline](https://aws.amazon.com/codepipeline/) and [AWS CodeBuild](https://aws.amazon.com/codebuild/) together form the Continuous Integration and Continuous Delivery services for Amazon Web Services (AWS).

The goal of this blog post is not to compare CodePipeline and CodeBuild with its competitors. It's simply an overview of both great and less great features that I've noticed since I've started working with them. I hope this overview can help you make an informed decision on whether CodePipeline and CodeBuild are the tools that make sense for your use case.

# The great

Let's start positive 

## Getting started

This is actually the reason that I started working with these tools in the first place. Assuming you are already working with AWS, getting started with CodePipeline and CodeBuild is a breeze. There is no external (SaaS) account to request, nor do you have to spin up any virtual machines on which to run an alternative CI/CD tool. There are no different tiers/plans to choose from: all features are immediately available once you have an AWS account. Pricing is very simple ([$1.00 per active pipeline](https://aws.amazon.com/codepipeline/pricing/), [pay for what you use with builds](https://aws.amazon.com/codebuild/pricing/)) which I prefer above paying ahead for a fixed amount of workers/agents/containers. Granted, with many pipelines the bill *will* go up, so definitely keep that in mind.

# The not so great

https://github.com/awslabs/aws-deployment-framework

## Cross-account and cross-region

# Conclusion