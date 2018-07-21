+++
title = "How to manage any kind of secret with AWS Secrets Manager"
author = "Sander Knape"
date = 2018-07-07T20:04:02+02:00
draft = false
tags = ["aws", "secrets manager", "lambda", "mongodb", "security"]
categories = []
+++
[AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) is a service recently released designed to make the management of secrets easier. It provides built-in support for Amazon RDS, making it very easy to set and rotate secrets and use the CLI or an SDK to retrieve secrets from applications. Through the use of custom Lambda functions, essentially any database or an otherwise protected endpoint is supported.  

Setting up Secrets Manager for a non-RDS database is less trivial as you need to write your own functionality using AWS Lambda. In this blog post we'll go through the process of creating a Lambda function for rotating a MongoDB user. First, let's dive in a bit more into what Secrets Manager can do for us.

# AWS Secrets Manager

The main feature of AWS Secrets Manager is secret rotation. Through a built-in Lambda function for the supported RDS services, or by writing your own Lambda function, you can schedule the rotation of a secret using a number of different strategies. Following a set of pre-defined steps which we'll cover in more detail later, Secrets Manager will set a new secret in the datastore or service, test it, and then store it in Secrets Manager so that an application can retrieve the latest secret to connect to the datastore.  

Your application can retrieve these secrets by using the AWS CLI or an SDK. Rather than storing a secret in your application configuration, instead you store a reference to a location within Secrets Manager where the application can retrieve the decrypted secret. Recently I wrote [a blog post concerning different options for retrieving secrets on deployment-time or run-time](https://sanderknape.com/2018/03/secret-management-design-decisions-theory-plus-an-example/), so be sure to check that out for more information regarding this. We'll dive more into the specifics of rotating the secret when we create the Lambda function later in this post.  

If this all sounds very familiar to you, you're right. All these features are basically a combination of other AWS managed services: Parameter store for storing and retrieving secrets, CloudWatch events for scheduling the secret rotation, AWS Step Functions for orchestrating the rotation and of course AWS Lambda. Secrets Manager therefore definitely isn't (yet) the service for every use case. Be sure to properly look into the service and keep in mind that if you need more flexibility, you can use a combination of the services I just mentioned. Another alternative is to use [Hashicorp's Vault](https://www.vaultproject.io/), arguably the best-known, open-source and self-hosted secret management tool out there.  

It's time to learn more about setting up a custom Lambda function. We'll start with configuring a test MongoDB instance, after which we build the Lambda function and configure Secrets Manager to rotate the passwords in MongoDB.

# Configuring MongoDB

**Important**: in this blog post we'll set up a MongoDB instance on EC2 that we'll open up **to the world**. This makes it a bit easier to connect to the database from our Lambda function. Normally though, be sure to spin up your database in a private subnet, and use the [Lambda VPC integration](https://docs.aws.amazon.com/lambda/latest/dg/vpc.html) to connect to the database from your Lambda function. A public MongoDB instance is definitely a bad practice: last year such open MongoDB databases [were often hijacked](https://www.bleepingcomputer.com/news/security/massive-wave-of-mongodb-ransom-attacks-makes-26-000-new-victims/). If you follow along with this blog post, keep in mind to kill your instance or to block the MongoDB port once you have finished playing around with it.  

As the goal of this blog post is not to teach you installing and configuring MongoDB, what follows is a concise list of steps to install and configure MongoDB. These instructions are for the Amazon Linux OS, but be sure to check out the [MongoDB installation guide](https://docs.mongodb.com/manual/installation/) for any other operating systems.

1. Start with spinning up an EC2 instance with a public IP address. Any other OS with a static, public endpoint will also suffice as long as you can install MongoDB on it. Write down the public IP address somewhere as we need it later. SSH into your instance.
2. Create the file `/etc/yum.repos.d/mongodb-org-3.6.repo` with the following contents:

```bash
[mongodb-org-3.6]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2013.03/mongodb-org/3.6/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.6.asc
```

3. Run the following commands to install and start MongoDB:

```bash
sudo yum install -y mongodb-org
sudo service mongod start
mongo
```

4. The last command opens the [mongo shell](https://docs.mongodb.com/manual/mongo/) which we'll use to finalize the initial configuration of MongoDB. First, let's run the following commands to be sure that we are properly connected to MongoDB:

```bash
use mydb
db.myCollection.insertOne({x:1})
db.getCollection("myCollection").find("1") # will return the just inserted value
```

5.  By default, MongoDB does not have authentication enabled. We will therefore first enable it and create the admin user and two application users:

```bash
use admin
db.createUser({user: "admin", pwd: "admin123", roles: [{role:"userAdminAnyDatabase", db: "admin"}]})
db.createUser({user: "app", pwd: "app123", roles: [{role:"readWrite", db: "mydb"}]})
db.createUser({user: "app_clone", pwd: "app_clone123", roles: [{role:"readWrite", db: "mydb"}]})
```

6.  Before we can use these new users, we need to enable authentication. Log out of the mongo shell using Ctrl+D. In `/etc/mongod.conf`, find the line with contents `#security` and change it to:

```yaml
security:
  authorization: enabled
```

7.  While we're in this file, find the line that says `bindIp: 127.0.0.1` and change this to `bindIp: 0.0.0.0`. This will allow us later to connect to MongoDB from the Lambda functions (please remember the security warning from earlier: we are opening up our instance to the world now)
8.  Next, run the following commands to restart MongoDB and login to the shell again:

```bash
sudo service mongod restart
mongo
```

9.  Let's login with the application user to see that we can indeed find the previously inserted value:

```bash
use admin
db.auth("app", "app123")
use mydb db.getCollection("collection").find("1") # again returns the previously inserted value`  
```
Great! Next, let's dive into AWS Secrets Manager and configure it to rotate the application user.

```
db.changeUserPassword("app", "app1234")
```

# Configuring Secrets Manager

The application user does not have the permissions to change its own password. This is in general of good best practice as it creates a separation of concerns where the application user can only do what it is supposed to do. Any administrative tasks such as changing passwords, grants or adding new users is then done by the administration user.  

In the following example we'll use the strategy of always having two users configured that can connect to the database (`app` and `app_clone`). Instead of (temporarily) causing authentication issues by changing an actively being used database user, we'll use a sort of ACTIVE/PASSIVE strategy where we rotate the credentials of the PASSIVE user and then tell the application to start using this user (essentially promoting the PASSIVE user to ACTIVE).  

To give an example: let's say that we set the TTL of a secret to 6 days. This means that an application will check in at least every 6 days to retrieve the latest secret. We also rotate the secret every week, but always keep the previous secret valid as well (the "previous secret" is then the PASSIVE user). Within 6 days the application will start using the new ACTIVE user. When we rotate the then-PASSIVE user again, by that time no application is still using that secret. All secrets are valid for 2 weeks using this approach, and we can be sure that we never change the credentials of a secret that is actively being used.  

Keep in mind that the "secret" is not just the password but also the username. The application will switch between using the `app` and the `app_clone` user. When requesting the ACTIVE secret from Secrets Manager, it will receive both the username and password.  

In Secrets Manager, the process of rotating a secret follows the next 4 steps;

1.  **Create secret.** In the first step, a new secret is generated and stored in Secrets Manager using the "PENDING" flag. Since your application should get the secret version with the "CURRENT" flag, the secret will not be used yet by your applications.
2.  **Set secret.** Next, the secret stored in the first step is retrieved and set in the actual database. From this point on, the secret is therefore actually configured in the database. If you do not use the multiple-user strategy, from this point on your application can not connect anymore.
3.  **Test secret**. The secret is then tested by, most likely, logging in to the database and possibly executing some specific queries to see if any grants/roles are configured correctly. If this step fails, it should reset the password to the previous one (which is still stored in Secrets Manager under the "CURRENT" flag).
4.  **Finish secret**. Finally, the "PENDING" secret in Secrets Manager is promoted to "CURRENT". The next time your application will retrieve the secret, it will receive the new URL.

If any Lambda invocation fails for some reason, the process stops; there is no rollback functionality. The whole process shouldn't take more than a few seconds, assuming the latency between the Lambda function and the database is OK. The Lambda functions are invoked synchronously but I haven't seen any delays in-between the invocations.

## Inserting the initial secrets

Let's begin with adding the secret values we configured in the previous section in Secrets Manager. Log in to your AWS account, open up the Secrets Manager console and click the "Store a new secret" button. Here, click the "Other type of secrets" button and insert the values for the admin account.  

![Create the admin user in AWS Secrets Manager](/images/secret-manager-admin-user.png)  

Click "Next" and use "mongodb-admin" as the name of the secret. Do not yet enable secret rotation. When the secret is created, open it in the console and copy the Secret ARN.  

![Get the ARN of the admin user in AWS Secrets Manager](/images/secret-manager-admin-user-masterarn.png)  

Next, add another secret for the application user. Again, select "Other type of secrets" and input the values from the screenshot below. This time we'll add the ARN of the master secret so that the Lambda rotation function will know to grab the values from the master secret to login to MongoDB to change the password. We also add the IP address of the MongoDB database.  

![](/images/secret-manager-app-user.png)  

Give this secret the name "monogdb-app". Again, don't enable rotation just yet.  

To get a better feeling for the Secrets Manager tool, you can play around with the AWS CLI to [describe the secrets we just inserted](https://docs.aws.amazon.com/cli/latest/reference/secretsmanager/describe-secret.html). Note how (by default) JSON is returned and that it should be really easy to automate running this command to inject the secrets in your application. As you can see, the username is also returned which is how the application will be able to switch between the users.

## Creating the Lambda function

Before we can enable rotation, we first need to upload the Lambda function that will perform the 4-step process to execute the secret rotation. First, create an IAM role that the Lambda function will use. Attach the `SecretsManagerReadWrite` managed policy and the `AWSLambdaBasicExecutionRole` policy to your IAM role (as always: be sure to give your Lambda functions only the minimal least-privilege permissions for non-testing functions!). Copy the ARN of the role you just created to your clipboard. It should look like the following:  

![Create an IAM role for AWS Lambda invoked by AWS Secrets Manager](/images/secret-manager-lambda-role.png)  

You will also need to download NodeJS (I'd recommend 8.x since we'll also use that version in AWS) through the [installation instructions](https://nodejs.org/en/download/). You can download the Lambda function code from my [GitHub repository](https://github.com/SanderKnape/aws-secrets-manager-custom-secret) and upload it through the following steps (make sure you have IAM credentials configured that can create a new Lambda function!).

```bash
git clone https://github.com/SanderKnape/aws-secrets-manager-custom-secret
cd aws-secrets-manager-custom-secret
npm install
zip -r function.zip *

aws lambda create-function \
--function-name mongodb-secret-rotation \
--runtime "nodejs8.10" \
--role "arn:aws:iam::xxxxxxxxxx:role/lambda-secret-manager-mongodb" \
--handler "index.handler" \
--zip-file fileb://function.zip

aws lambda add-permission \
--function-name mongodb-secret-rotation \
--principal secretsmanager.amazonaws.com \
--action lambda:InvokeFunction \
--statement-id SecretsManagerAccess
```

The Secrets Manager service is still relatively new, so be sure that your AWS CLI version is up to date. We first create the deployable package by downloading the dependencies and creating the ZIP file. We create the function, and finally give the Secrets Manager service permissions to invoke the Lambda function.  

The Lambda function shows the bare minimum for creating a custom rotation function. For brevity, I've only added the "happy path" flow. Check the [built-in Lambda functions](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html) from AWS to see what kind of extra checks you can perform to increase the reliability of the function. This will also ensure that retries will work without errors.  

**Important**: your Lambda CloudWatch logs will be filled with old and new secrets for debugging purposes. For real-life scenarios, be sure not to log your secrets!

## Enabling rotation

We can finally enable rotation for the application user in MongoDB. Open up the Secrets Manager console and open the "monogdb-app" secret. Here, select "Edit Rotation" and enable rotation. Select the Lambda function "mongodb-secret-rotation" which we just created.  

![Enable secret rotation in AWS Secrets Manager.](/images/secret-manager-enable-rotation.png)  

After enabling rotation Secrets Manager will immediately start the rotation. Within a few seconds you will be able to see the new secret by clicking the "Retrieve secret value" button. Also note that not only the password changed but the username as well.  

![We have successfully rotated the secret in AWS Secrets Manager.](/images/secret-manager-finished.png)  

Be sure to add some extra logging to the Lambda function if you want to follow along more what is going on!

# Conclusion

In this blog post I gave a small introduction to AWS Secrets Manager and went through the process of setting up a custom secret including rotation with a custom Lambda function. The service currently is still very lightweight, and can definitely not compare feature-wise with an alternative such as Hashicorp's Vault. However, Secrets Manager is a fully managed, serverless solution that is sure to receive more features in the future.  

Would I recommend AWS Secrets Manager? If you're familiar with AWS Lambda, I would definitely recommend the service for small, simple use cases such as the one I demonstrated in this blog post. Rotating secrets for RDS is of course built-in, so I'd definitely recommend the service if you want to rotate RDS passwords. If you just want to just store secrets and don't care about rotation, you can simply use [AWS Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html) which is much cheaper.  

I'm certainly interested in your opinion regarding this service and how it compares with other services you might use. Let me know in the comments!
