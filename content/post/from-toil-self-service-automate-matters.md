+++
title = "From toil to self-service: automate what matters"
author = "Sander Knape"
date = 2020-06-22T11:44:02+02:00
draft = false
tags = ["autonomy", "self-service", "automation"]
categories = []
+++
There are a few reasons that I love my job. One of the most important ones is the variety of work. As a cloud/platform engineer, every day is different. Work goes from writing automation in some programming language, setting up a dashboard in a monitoring/logging tool, hardening Linux machines, writing Infrastructure as Code, building (standardized) CI/CD pipelines, giving workshops, analyzing costs, and more.

This wide variety of work wouldn't be possible without automation. You have more time to spend on all these things when manual, repetitive work is automated. SRE [defines toil](https://landing.google.com/sre/sre-book/chapters/eliminating-toil/) as follows:

"*Toil is the kind of work tied to running a production service that tends to be manual, repetitive, automatable, tactical, devoid of enduring value, and that scales linearly as a service grows*."

I don't mind toil per se. What I care about is that because I haven't automated something, someone else now needs to wait for me to do some manual work. I'm essentially blocking the software development lifecycle of my colleagues. Furthermore, as part of the definition of toil, this work also scales linearly. So with more services created and the more developers that join the company, more manual work is required from me.

{{< figure src="/images/self-service.png" class="right" >}}

I wrote before about some ways to [provide autonomy to developers](https://sanderknape.com/2019/07/five-ways-enable-developer-autonomy-aws/) (in that post, specifically in AWS). I definitely believe that providing autonomy is possible by including boundaries for security/costs in automated processes. Instead of performing manual work to do something for a colleague, you can offer this required capability as an automated service: self-service. Without your help, without you "blocking" progress, this self-service capability enables a colleague to get it done autonomously.

Building automation of course takes time. Rome wasn't built in a day. Certain tasks take priority to work on and some manual work (toil) may remain. It can take years before a fully automated platform is build on which development teams can autonomously build, deploy and operate their applications. And more importantly: whether something is automated or not isn't a binary "yes it is" or "no it isn't". You can partly automate a task and provide the desired result more quickly.

It can be valuable to automate toil in steps and focus on specific parts of the task first. In this post, I'll describe how I typically work towards providing self-service while attempting to find the balance between delivering short-term value and keeping the long-term goal in mind.

## Towards self-service

In my experience, there are four steps that you typically go through when iteratively replacing toil with self-service. Let's go through these step by step.

### 1. Embrace toil

The first step to getting rid of toil is through embracing it. Take your time to do the work. Be mindful of it. Ask yourself some questions. Is this work actually important? Could I perform this toil in fewer steps? Are there other processes that are related to this? How does what I'm doing fit into the larger scheme of things?

There is perhaps more value than you initially realize in executing this toil. It teaches you a lot about what you are doing and why you are doing it. So keep asking questions. You'll probably learn a lot about similar processes and perhaps even how these are (not) automated. You may see how this work should be part of some other job as that would be more efficient. Or you realize that this work should be split into multiple separate tasks, as some of it is only needed in particular situations.

This is about the right time to ask yourself if it's worth automating this work. You have now done the job multiple times and you are convinced of the value it brings (or not, and you stopped doing it). You also went through some iterations to improve the process and you have perhaps merged or split it. You probably know by now how often this work has to be performed.

If you have decided to automate this toil, it's time to start thinking about how that would work. Is there an API for the tool you're using? Are there any other tools that would make automation easy? What other tools are used in the company to automate similar work?

Once you are there, it's time for the next step.

### 2. Automate the first steps

It is worthwhile to see if what you want to automate can be done in smaller steps. While this may not make sense for some simple tasks (create a Git repository with default settings), other jobs may require a lot of steps (create an Azure Subscription, configure it with default resources, add a VNet, peer it to another network, create an Azure DevOps pipeline using standardized Terraform templates). If your toil requires multiple different steps, you could start with automating just a part of it.

You could first automate steps 2 and 4 and still manually perform steps 1, 3 and 5. You may have just cut down the time to get this whole process done by 95%. This means that you were able to provide some short-term value while you still have the end goal in mind. You may even decide that this is enough and that no further automation is needed.

What's important is that you now have room to experiment. Only you and your team members are now using this automation. The original requester isn't even aware of this other than the fact that perhaps the requests suddenly get delivered faster. Because the end-user isn't using your automation (yet), you can still take the time to improve the automation while you already benefit from it. Perhaps you automated steps 2 and 4 using two completely different tools and you still need to stitch them together. Each time that you have to execute this process again, its a chance to improve or extend the automation further based on new use cases that you perhaps haven't thought of. You may even start looking forward to another opportunity to try it all out.

Keep in mind that you might want to make your automation idempotent. Perhaps it should be required to rerun the script to configure specific settings correctly. Maybe you already know that this is going to be crucial when you provide this automation as a self-service mechanism to the end-users. Now is a great time to add idempotency.

### 3. Bring it all together

The third step is about bringing it all together. The goal here is to simplify your (possibly: set of) automation scripts to a single entry point. At the end of this step you should be able to execute a single script that fully gets the task done. You may need to write some glue code that brings different pieces together. Some steps may take a while to run; some steps may have dependencies on other steps. You may want to look into some tools to ease building these kinds of workflows, such as [Apache Airflow](https://airflow.apache.org/) or [AWS Step Functions](https://aws.amazon.com/step-functions/).

Fully automating a complex workflow is bound to contain a bug or two initially. That's why still keeping the automation to yourself first can be valuable. You may run into race conditions as you learn of dependencies between steps that you weren't aware of before.

And don't forget: your end-users already benefit from tickets getting resolved faster. And you benefit because you're working on a (hopefully) fun project. You can freely make further improvements and experiment with different setups.

While the main goal is to get everything together, it's also essential to keep the next step in mind: provide this automation through self-service. How would you provide this automation to the end-user? Is another tool already used that you could use? Which parameters should be passed by the end-user and which should be built-in or resolved dynamically as part of the task?

### 4. Self-service

The final step is to wrap everything you have already build into a self-service mechanism for your end-user. Because you have already automated the entire workflow, this step is mainly about usability, access and insights.

#### Usability

You are now building a service for your end-users. They will want a user-friendly way to interact with your service. It very much depends on your end-user what is a good experience for them. Business users probably prefer an attractive user-interface. Developers may prefer a YAML file that they commit to a Git repository. If there are already tools used in the company for making requests, it may make sense to make the service available through that tool.

You also want to help them with properly making the request. A set of parameters must probably be provided to be able to execute the automation. If one of the parameters is a boolean, present a radio button in the UI. If a number must be specified that has a minimum and/or maximum value, ensure only those values can be entered. If a request is made through a YAML file in a Git repository, run automated checks in the pull/merge request that provides explicit feedback if a parameter is missing or incorrect.

The value here is that your end-user will be able to execute the automation autonomously. Lacking usability means that they would still have to come to you to ask why the automation doesn't accept a request. Write documentation to explain how the self-service must be used and describe (opinionated) decisions made in the workflow. It's not proper self-service if only the happy path is accounted for.

#### Access

Self-service doesn't have to mean that everyone can just trigger the workflow. In many cases only certain people or roles must be able to request or approve the workflow. The *business flow* is thus kept intact (or even improved), but the *technical flow* is automated away and no longer blocking for executing the request. When you investigate which tooling to use to provide self-service, consider how to implement this approval flow.

I'm personally a fan of making requests through Git. All changes to the repository ("requests") are logged by default. Through repository access you can exactly specify who can make requests and who can approve (merge) request. In my experience, this is a simple method for platform/cloud/DevOps teams who receive requests from development teams. Again: it depends on what your end-user is already familiar with. Try to give them an experience that is consistent with what they already know.

#### Insight

Things still can go wrong and when they do, you should get notified. Integrate your automation with existing logging & monitoring tools within your company to have insights into its state. Set up alerts in case your end users get errors in your self-service mechanism. You'll want to learn about issues through alerts before your end-users tell you about them.

While some people may need to approve the workflow, other people may simply need to be informed that a workflow has been executed. Perhaps a manager is allowed to approve a firewall change but the security department must receive a notification about that change. Keep usability in mind here as well: the security department may want to see all configured rules on that changed firewall easily. Provide dashboards that give real-time insight into the state of resources that can be altered through self-service.

## Conclusion

Companies always want to go faster. Self-service is a great way to provide autonomy to other teams who would otherwise need your help to get something done. But be pragmatic: self-service isn't the goal here. Spending weeks on automating something that is done a few times per year and takes 5 minutes isn't effective nor efficient. In my experience, following the four steps in this post helps me automate what makes sense. I hope this little framework will help you in making decisions as well.
