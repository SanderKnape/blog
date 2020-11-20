+++
title = "Nuances around centralized platform teams"
author = "Sander Knape"
date = 2020-11-20T13:49:11+02:00
draft = false
tags = ["platform engineering", "autonomy", "devops"]
categories = []
+++

The popularity of centralized platform teams is rising. The latest [Puppet State of DevOps Report](https://puppet.com/blog/2020-state-of-devops-report-is-here/) shows that 63% of the respondents have at least one internal platform. Platforms are vital enablers for a more DevOps way of working as they provide self-service capabilities that development teams can autonomously utilize.

The definition of a "platform" isn't set in stone though. Many organizations still struggle to put together a platform team that is really able to add value to the development teams. It's a challenge to build a team with the proper mindset and an organization that supports that team in the right way. The biggest challenges aren't technical: it's the organizational and cultural challenges that must be tackled to ensure such a team's effectiveness.

A certain pragmatism is required when implementing a platform team in an organization. A platform team is known to provide centralized capabilities to its development teams. But as with everything, in practice it's not so clear what those capabilities are, which capabilities to take ownership of and who makes the decisions. In this post I'll share some nuances around the practicalities of building centralized platform teams.

## What's in a platform?

A platform is an integrated set of tooling, abstractions and processes on which developers can build, deploy and operate their applications and take end-to-end ownership of their applications.

In general I consider four categories of capabilities that an internal platform must provide. The first category is a set of **Fundamentals** such as a cloud environment (an AWS account, Azure subscription, a Google Cloud project), Identity and Access Management (ensuring development teams have access to their environments and only their environments), and Networking (allow applications to privately and securely connect).

The second category is **Delivery**. With these fundamentals in place, development teams must now have the capabilities to deploy their applications to their environments. Their CI/CD pipelines need least-privilege access to only their environments and many capabilities can and must be provided around these pipelines primarily around standardization and automated testing.

Third, many **Insights** are required for the development teams to take ownership of their applications properly. Such insights include operational insights around logging, monitoring and metrics as well as cost insights and security insights.

Finally, **Security** is a whole category out of itself. Apart from the security insights, all the platform's capabilities must be implemented and provided with security at the top of mind. Additional security-specific capabilities must be provided as well such as automated penetration testing, static code analysis, DDOS protection, infrastructure compliance checks and many more.

The primary value brought by the platform however is the glue between all these different capabilities. All networks created in the cloud environments must be securely connected to allow traffic between the applications. The Identity and Access Management (IAM) capabilities of the cloud environment are typically connected to a central IAM system to provide least-privilege access to the development teams. Monitoring and logging systems should be configured to gather relevant logs from the environments automatically. The CI/CD systems must be hooked on to the IAM capabilities to deploy infrastructure and applications to the environments. The list goes on and on.

Cliches are cliches because they are true, right? *The platform is greater than the sum of its capabilities.*

One of the primary goals of a platform is to abstract/automate away complexity from developers. Most development teams aren't equipped with networking expertise. A platform that abstracts away networking therefore significantly reduces the *cognitive load* for these teams: the development teams can focus more on building their applications. Besides, centrally managing networking also makes it easier to construct compliant networks (routing all traffic through centralized firewalls, for example) and makes it easier to prevent overlapping CIDRs. However, there are organizations where developers are expected to take care of the networking part themselves.

Which brings me to the first nuance: the capabilities provided by an internal platform will be different for each organization. It's up to each organization to learn what brings the most value. If development teams are already equipped with networking expertise, they may not appreciate a centralized platform team taking over this responsibility.

There is no clear-cut template for what exactly entails a centralized platform. Treat your platform as a product where the development teams are your customers. Speak with your customers to learn what capabilities they require, prioritize, and get started.

## A platform team doesn't have to be a single team

A platform team is just another development team. As the platform grows, the expertise required by the platform team grows with it and there may come a point where the knowledge required is too much for a single team. A platform provides value by lowering the cognitive load for development teams. It's easy to forget to also keep an eye on the cognitive load of the platform team itself.

Once the platform team's responsibilities become too broad or when the team becomes too big, look at what capabilities are easiest to split across different teams. Keep tightly integrated capabilities within a single team. Identify which capabilities are more loosely coupled and find a set of capabilities that can most easily be extracted and moved to a new platform team.

## Development teams can be platform teams

Providing centralized capabilities shouldn't be limited to only the platform team. It's perfectly fine when a development team takes ownership of a capability and provides it to other teams within the organization.

Let's say a development team requires the capability to send out large amounts of e-mails. This isn't something yet available within the organization. They decide to go with a SaaS product which provides this and soon other development teams start asking for the same capability.

You may argue that now that more teams require this capability, the platform team should take ownership of this tool. But why? The development team that picked the SaaS product is perfectly capable of enabling the other development teams to start using it. They set up some rules (naming conventions, a schedule for when to sent out e-mails) and work with other teams to get them onboarded onto the tool. An added benefit is that this type of communication fosters collaboration between the development teams. And they get more experience with providing (self-service) capabilities to other teams, perhaps getting a better understanding of the work the platform team does to enable capabilities in a usable, self-service way.

Don't enforce that centralized capabilities must always be provided by a platform team.

## The platform team doesn't need to cover 100% of the use cases

A platform team provides standardized, re-usable capabilities that development teams can take advantage of. An inherent result of this approach is that these capabilities will be opinionated. Many of the complexities are abstracted away in the standard way of providing this capability. Therefore, the capability is less flexible, making it harder - if not impossible - to use by use cases that deviate from the standard way of working.

A platform team can't cover 100% of the use cases requested by the development teams. Let's say that the platform team provides standardized deployment pipelines for dozens of development teams that deploy applications to Kubernetes. There is also one mobile development team that doesn't deploy to Kubernetes but to a mobile app store. The platform team could extend the current deployment capabilities to also include this use case.

However, expertise around building and deploying mobile applications is most likely largely available within the mobile development team. Those developers are likely already experienced with these processes and are aware of tooling and automation that can help them achieve this. Why not let the team take care of such capabilities themselves?

You might even say the mobile development team is more "DevOps" than the other development teams. Because they fully control their own deployment pipelines, they are less dependent on an external factor (the platform team) in regards to their automation and tooling.

Be flexible in when there is added value in providing a capability from a centralized platform team. If dozens of teams need to investigate how to deploy to Kubernetes, a simple, standardized solution provides a lot of value. But if a team requires a unique capability, it's perfectly fine to make them responsible for choosing, building and maintaining their own tooling.

## A platform team isn't the police (mostly)

The centralized platform team doesn't work in isolation. They collaborate with other roles within the organization to build a platform that adds value. For example;

* The platform team experiments together with developers to build standardized deployment pipelines. Together they decide which tooling to use and what kind of sane defaults to implement
* The platform team collaborates with the Security department to ensure the environment is compliant. Certain restrictions are built into the platform. For example, encryption in transit and at rest must always be enabled for relevant cloud resources
* The platform team collaborates with architects to ensure the applications and infrastructure are built according to best practices. Architects may decide that backups must always be enabled for databases to stipulate proper Recovery Time and Recovery Point Objectives (RTO and RPO)
* The platform team learns from the Finance department that the environment costs must be allocated to the correct development teams. To provide this information, tagging of all cloud resources is made mandatory using a defined, standardized format

One of the challenges around autonomous development teams is how to stay in control. A practical way to ensure this is to implement certain restrictions into the platform such as the previous examples. Such restrictions are often defined not by the platform team itself but by other roles within the organization. In those situations, the platform team is "just the messenger". If it's not clear who imposes certain restrictions within the platform, the perspective may originate that the platform team makes all the decisions. This can undermine the relationship between the platform team and the development teams.

Of course, the platform team itself may also decide to built in certain restrictions, largely to *protect the platform* and, consequently, the users of that platform.

Take [concurrency limits with AWS Lambda](https://aws.amazon.com/about-aws/whats-new/2017/11/set-concurrency-limits-on-individual-aws-lambda-functions/) as an example. Each Lambda function can be configured to "reserve" a part of the total concurrency for all Lambda functions. If one or more Lambda functions together request the total concurrency available, all other Lambda functions will stop working within the AWS account.

When multiple development teams build applications in a single AWS account, it's hard to manage all individual requests for limiting concurrency per function. The platform team may decide to simply block this functionality to ensure the platform and its users are protected from any issues caused by this functionality. (This is, by the way, a reason to go for a multi-account strategy)

Realize that platform teams don't work in isolation. It's easy for roles such as security, architects or finance to "hide" behind the platform teams. Ensure such roles are clearly visible within the company and that these roles advocate why certain decisions are made within the platform. Providing such transparency makes it easier for developers to get in touch with the correct people when they require more information about the restrictions.

## Conclusion

There is no clear definition of how a platform team should be composed and what exactly such a team should be delivering. Be transparent about the team's goals and continuously validate with stakeholders if what is delivered is still working towards that goal. Keep an open mind for what exactly brings value to your customers (the development teams). Don't take ownership of capabilities just for the sake of taking ownership. Provide centralized capabilities where it makes sense, and let the development teams take care of the rest themselves.
