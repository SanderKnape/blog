+++
title = "Installing private Git repositories through npm install in Docker"
author = "Sander Knape"
date = 2019-06-17T13:30:02+02:00
draft = false
tags = ["docker", "git", "npm", "ssh", "buildkit"]
categories = []
+++
How do you properly use an SSH key in a Dockerfile? There are many ways to do it, including many ways to do it wrong. What you will want to prevent is that your ssh key ends up in one of your intermediate images or layers. These are the layers that Docker creates with pretty much every command in your Dockerfile. You may think that you properly clean up your secrets later in the Dockerfile, but the secret will then still be available in one of these layers.

This is especially problematic when you build your Docker images in a (SaaS) CI/CD tool that supports caching. As the cache is uploaded to the system of your provider, it may very well happen that your secret ands up plain-text on their servers.

If you want to learn more about these layers, be sure to check out this [great post that explains much more](https://medium.com/@jessgreb01/digging-into-docker-layers-c22f948ed612).

How then do you properly use secrets in your Dockerfile? In this blog post we'll look into a common use case: downloading private git repositories through an `npm install`. We'll dive into two different methods to tackle this in a way that we not expose our secrets in our Docker layers.

In this post I'll use a private repository on GitHub as an example. Any other git provider will however also work with this approach.

Let's get started!

## Prerequisites

Let's first get some prerequisites set up before we dive into the two methods.

### Private GitHub repository

Testing how to install a private git repository is of course pretty hard without a private git repository. Therefore, make sure you spin one up if you don't already have one. Keep in mind that you can create a free private repository on GitHub since [the beginning of this year](https://github.blog/2019-01-07-new-year-new-github/).

Also, be sure this repository at least contains a `package.json` file with at least an empty object `{}` as its contents. Otherwise the `npm install` that we run later won't recognize this is a valid NPM repository.

### SSH key

You will also need an SSH key that you can use to download the private repository from GitHub. And, let's be sure that this actually works on your laptop: otherwise you may be debugging something in Docker that doesn't even work directly on your laptop!

Check out the GitHub documentation on [how to add an SSH key to your account](https://help.github.com/en/enterprise/2.15/user/articles/adding-a-new-ssh-key-to-your-github-account). Also, be sure to [add the key to your ssh agent](https://help.github.com/en/enterprise/2.15/user/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent). The easiest way to do this is by adding the following configuration to the `~/.ssh/config` file:

```
Host github.com
  Preferredauthentications publickey
  IdentityFile ~/.ssh/github
```

Be sure to replace the name of the key with the name that you have choosen.

You should now be able to clone your private repository through SSH with the following command:

`git clone git@github.com:SanderKnape/ssh-test`

Of course, change the repository owner and name to your own private Git repository.

### An NPM project that has a dependency on the private repository

You need a NodeJS project with a `package.json` that has the private git repository as a dependency. At the minimum you must have a `package.json` with the following contents:

```json
{
    "name": "ssh-test",
    "version": "0.0.1",
    "description": "Testing installing a private repository",
    "dependencies": {
        "my-ssh-test-dependency": "git+ssh://git@github.com/SanderKnape/ssh-test.git#master"
    }
}
```

Again, be sure to change the git repository to your own. Now, run `npm install`, and the repository should succesfully download.

With all of this setup, let's see two different methods for how we can install this private git repository with `npm install` in a Dockerfile.

## Docker Buildkit

The first method uses the [Docker Buildkit](https://docs.docker.com/develop/develop-images/build_enhancements/). This is a relatively new way for building Docker images with advantages such as better performance and more features. One of those new features is the `--ssh` flag, which allows you to forward your SSH agent to the Docker container. What is great is that no keys are copied to your Docker image. During the SSH connection, Docker simply uses your local SSH agent which keeps your key in memory. There is therefore no way for your key to end up in one of your Docker layers.

These new features are currently available under an `experimental` flag. We are going to create a Dockerfile in the same location as the `package.json`, and build this Dockerfile with the following command:

`DOCKER_BUILDKIT=1 docker build --ssh github="$HOME/.ssh/github" -t ssh-test .`

First of all, the `DOCKER_BUILDKIT=1` flag enables the use of the new features. You can also add the following to the `/etc/docker/daemon.json` file so that you don't always need to type this part:

```json
{
    "features": {
        "buildkit": true
    }
}
```

Next, the `--ssh github=$HOME/.ssh/github` part tells the Docker builder that it is allowed to use that key. Let's see how that looks like in the Dockerfile. Create a new file called `Dockerfile` in the same location as the `package.json`.

Note the first line in the following Dockerfile that is also required to make use of this feature:

```Dockerfile
# syntax=docker/dockerfile:1.0.0-experimental

FROM node:10-alpine

RUN apk add git openssh-client

COPY package.json ./

RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts

RUN --mount=type=ssh,id=github npm install
```

Especially interesting is the `RUN --mount=type=ssh,id=github npm install`. Here we tell Docker that it is allowed to use the `github` key that we passed on `docker build`. This key is then available to the Docker builder as it connects to the local SSH agent, which sees in the `~/.ssh/config` file that it must use this key to connect to the GitHub server.

And that's it! Docker BuildKit comes with some great performance boosts and additional features. Be sure to check out the [official documentation](https://docs.docker.com/develop/develop-images/build_enhancements/)for the other features that are now available.

## Fallback method

If you are still on an older version of Docker, or when you are not comfortable using `experimental` features, the following approach is the second-best thing. We'll forward the SSH key to to `docker build` on the CLI, but never persist the key on disk to prevent the secret from leaking.

Replace your Dockerfile with the following contents:

```Dockerfile
FROM node:10-alpine

ARG SSH_KEY

RUN apk add git openssh-client

COPY package.json package-lock.json ./

RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts

RUN ssh-agent sh -c 'echo $SSH_KEY | base64 -d | ssh-add - ; npm install'
```

With `ARG SSH_KEY` we tell the builder that an environment variable will be passed to the `docker build` command that must only be available during build time. The other lines are all exactly the same, except for the last line where we use the `SSH_KEY` variable that we pass. We create a subshell where we first pass the `SSH_KEY` to `ssh-add` (we will actually pass a base64-encoded string to the Docker builder, so we decode it first). We do this using `standard input`, which means that we do not need to persist this key to disk, and the key is only kept in memory. This way, again the key will not leak to one of your cached Docker layers.

We now run `docker build` a little different. First, create a base64-encoded version of your key as follows:

```bash
key=$(cat ~/.ssh/github | base64)
```

Now, we pass this key to the Docker builder:

`docker build --build-arg SSH_KEY=$key -t ssh-test .`

Run the command and see your private git repository properly being installed. Run the command again and see how caching nicely speeds up your build.

## Conclusion
Always be careful not to store any secrets in your Docker layers. In this post I presented two different ways of safely using an SSH key in your Dockerfile. I would definitely recommend to try out to the BuildKit approach, as it brings some additional features and performance improvements as well.



