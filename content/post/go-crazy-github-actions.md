+++
title = "Go crazy with GitHub Actions"
author = "Sander Knape"
date = 2021-01-13T16:32:12+02:00
draft = false
tags = ["github", "actions", "ci/cd"]
categories = []
+++

[GitHub Actions](https://github.com/features/actions) is a component of GitHub that allows you to create automated workflows. Through the many different [events](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows) that can trigger workflows you are free to build whatever automation you want. While the most common use case is building CI/CD pipelines, the possibilities are pretty much endless. Check out this list of [awesome actions](https://github.com/sdras/awesome-actions) to get some inspiration.

Having spent quite a bit of time with GitHub Actions in the last few months I came across some features that aren't very well documented. It's therefore very well possible that not everyone is familiar with these capabilities. Let's dive into five neat features that you can go crazy with.

## If statements

From a first glance at the documentation it may seem there is no support for if statements in expressions (other than deciding [whether or not to run a step/job](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#example-expression-in-an-if-conditional)). But, if statements do exist. Take a look at the following workflow:

```yaml
name: If

on: push

jobs:
  if:
    name: If
    runs-on: ubuntu-20.04
    strategy:
      matrix:
        job: [a, b]
    steps:
    - run: echo ${{ (matrix.job == 'a' && 'foo') || 'bar' }}
```

Job `a` will output `foo` whereas job `b` will output `bar`. You can read the expression as the following if statement (pseudo code):

```
if (matrix.job == 'a') {
  echo 'foo'
} else {
  echo 'bar'
}
```

This is even more powerful when combined with one of the [functions](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#functions) or [operators](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#operators) available. You can for example check the value of a number:

```yaml
name: If

on: push

jobs:
  if:
    name: If
    runs-on: ubuntu-20.04
    strategy:
      matrix:
        node: [4, 6, 8, 10, 12]
    steps:
    - run: echo ${{ (matrix.node > 8 && 'foo') || 'bar' }}
```

If the syntax confuses you, check out [short-circuit evaluation](https://en.wikipedia.org/wiki/Short-circuit_evaluation). To summarize: an AND operator returns `true` only if both arguments are true. If the first argument is not true, there is thus no need to evaluate the second argument as it doesn't matter whether it's true or not.

It works very similarly for the OR operator. As it returns `true` if at least one of the arguments is true, there is no need to evaluate the second argument if the first one is already true.

This behaviour thus "short-circuits" the evaluation of the passed arguments to a boolean operator. The combination of these two operators is what gives us the `if` functionality in GitHub Actions.

## Composing secret names

You may be pre- or post-fixing your secrets in certain situations. For example. when working with [AWS access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey) for different environments, you may name these keys like this:

* For dev: `DEV_AWS_ACCESS_KEY_ID` and `DEV_AWS_SECRET_ACCESS_KEY`
* For prod: `PROD_AWS_ACCESS_KEY_ID` and `PROD_AWS_SECRET_ACCESS_KEY`

In your workflow you will then compose the secret's name based on the environment you want to deploy to. The easiest way to do this is by using the [format](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#format) functions like in the following example:


```yaml
name: Secrets

on: push

jobs:
  deploy:
    name: dev
    runs-on: ubuntu-20.04
    strategy:
      matrix:
        env: [dev, prod]
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets[format('{0}_AWS_ACCESS_KEY_ID', matrix.env)] }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets[format('{0}_AWS_SECRET_ACCESS_KEY', matrix.env)] }}
    steps:
      - run: aws s3 ls # just something random to see that authentication works
```

Note this also works with other expressions such as with the `workflow_dispatch` event:

```yaml
name: Secrets

on:
  workflow_dispatch:
    inputs:
      env:
        description: "Environment to deploy to"
        required: true

jobs:
  secrets:
    name: secrets
    runs-on: ubuntu-20.04
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets[format('{0}_AWS_ACCESS_KEY_ID', github.event.inputs.env)] }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets[format('{0}_AWS_SECRET_ACCESS_KEY', github.event.inputs.env)] }}
    steps:
      - run: aws s3 ls # just something random to see that authentication works
```

Recently GitHub made it possible to set secrets [per environment](https://github.blog/changelog/2020-12-15-github-actions-environments-environment-protection-rules-and-environment-secrets-beta/) which is a step in the right direction, but this doesn't work (yet) with organization-wide secrets. Use this method when using environment secrets isn't an option for your use case.

## Dynamic environment deployments

A quick tip about the new [environment feature](https://github.blog/changelog/2020-12-15-github-actions-environments-environment-protection-rules-and-environment-secrets-beta/) that I just mentioned. I wanted to implement manual deployments using these new environments using the `workflow_dispatch` event. I also tried to deploy multiple environments using a `matrix`. However, setting the environment to a dynamic value gave me an error - it looked like environments only support static values.

Luckily, the GitHub Community came to the rescue. I created a [new topic](https://github.community/t/environment-name-does-not-support-matrix/150331) on the GitHub Actions forum and one of the replies mentioned that dynamic values are indeed possible.

The environment property has a "short" and "long" form. Dynamic values only work with the "long" form. The following is a fully working example:

```yaml
name: Environment

on:
  workflow_dispatch:
    inputs:
      env:
        description: "Environment to deploy to"
        required: true

jobs:
  secrets:
    name: secrets
    runs-on: ubuntu-20.04
    environment:
      name: ${{ matrix.environment }}
    steps:
      - ...
```

## Private actions

The most powerful component of GitHub Actions is the set of [Actions](https://github.com/marketplace?type=actions) that can do pretty much anything as part of your workflow. Through these actions, you can create workflows with a lot of logic and minimal code duplication.

It's unfortunate therefore that you can only use publicly available actions. It's interesting for organizations to build custom actions that they don't want to share with anyone outside of the company. At the time of writing, the [GitHub public roadmap](https://github.com/github/roadmap/issues/74) shows that this feature is planned but not before the end of Q2 2021.

A workaround is available luckily, though a little work is required to get it up and running and it's not without its downsides.

1. **Create a Personal Access Token**. We first have to work around another missing feature of GitHub. The default access token available within a workflow only allows access to the repository that owns that workflow. This means that we can not use this token to fetch another repository's contents within the same organization. It's not possible to [extend this scope](https://github.community/t/github-actions-token-scope/17061). You will therefore have to create a Personal Access Token with at least the `repo` scope that has access to other repositories within the organization. **Important**: this access token will have access to **all** repositories. If this is a problem you can consider using a [Deploy key](https://docs.github.com/en/free-pro-team@latest/developers/overview/managing-deploy-keys#deploy-keys) which grants access only to a specific repository. This requires more work to setup though which I don't cover here. Also, note that the personal access token is linked to the user who creates it. When that user leaves the organization the access token will stop working. Anyway, store this access token as a secret within the repository (or on the organization level to easily share it with multiple repositories)
2. **Download the repository with the private action as part of your workflow**. In our workflow (you can find an example below) we will first download the contents of the repository with the private GitHub Action. The easiest way to do this is by using the [checkout action](https://github.com/actions/checkout) which also support specifying a different repository.
3. **Execute the just downloaded private action**. Now that the action is available on the file system, we can execute it. This is possible because the GitHub [uses](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepsuses) property also supports specifying a location on the file system.

The following is an example workflow that executes a private action:

```yaml
name: Private Action

on: push

jobs:
  action:
    name: Action
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout this repository
        uses: actions/checkout@v2
      - name: Checkout actions repository
        uses: actions/checkout@v2
        with:
          repository: [owner]/[repository]
          token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
          path: .private-actions/
      - name: Execute my action
        uses: ./.private-actions/my-action
```

In this example I'm assuming that the actions repository contains a subdirectory `my-action` with the actual action. This is how you can store multiple custom actions in a single repository. It's also possible of course to create a seperate repository per custom action.

Note that the checkout action also supports specifying the branch, tag or SHA to checkout. You can therefore version your private actions similarly to how public actions are versioned. If you tag your commits with `v1`, you can add `ref: v1` to the `Checkout actions repository` step.

## Trigger on pull request merge

There are many different [events](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows) available on which you can trigger your GitHub Actions workflows. A notable absent event however is to trigger a workflow when a pull request is merged. This is interesting for various use cases such as integrating with a ticketing system or tracking some KPIs around your pull requests (such as how fast they are merged).

Of course, under the hood the merge of a pull request is a push to whichever branch the pull request was submitted. However, a push to a branch doesn't guarantee that it was pushed "through" a pull request. Currently, there is no way to tell if a `push` event is triggered through a pull request.

Another option is to use the [pull request closed](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows#pull_request) trigger, but this would also trigger if a pull request is closed but not merged. Luckily, GitHub provides additional information in the [context objects](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#contexts). We can leverage this information to identify whether or not the workflow was triggered by merging a pull request when a pull request is closed.

```yaml
name: PR

on:
  pull_request:
    types: [closed]

jobs:
  merged:
    runs-on: ubuntu-20.04
    if: github.event.pull_request.merged == true
    steps:
      - run: echo "MERGED!"
  not-merged:
    runs-on: ubuntu-20.04
    if: github.event.pull_request.merged == false
    steps:
      - run: echo "NOT MERGED!"
```

Test this workflow by either closing or merging a pull requesting. You will notice that only one of the jobs will trigger.

![](/images/github-actions-merge-event.png)

Note that you can also [filter on specific branches](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows#example-using-multiple-events-with-activity-types-or-configuration) to only trigger after a merge to a specific branch.
