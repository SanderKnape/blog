+++
title = "Improving Kubernetes deployments with Helm"
author = "Sander Knape"
date = 2019-03-15T22:47:02+02:00
draft = false
tags = ["kubernetes", "helm", "configmap"]
categories = []
+++

I recently blogged about [automated deployments to Kubernetes using GitLab](https://sanderknape.com/2019/02/automated-deployments-kubernetes-gitlab/). One of the steps required when automating deployments is replacing the Docker tag with the correct value in the Kubernetes Deployment. In that blog post, this looks like the following:

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  [...]
spec:
  template:
    spec:
      containers:
          image: sanderknape/go-hello-world:<VERSION>
  [...]
```

The `<VERSION>` string is then replaced in the GitLab pipeline as follows:

`sed -i "s/<VERSION>/${CI_COMMIT_SHORT_SHA}/g" deployment.yaml`

This grabs the short SHA hash of the current Git commit that is checked out. Earlier in the pipeline, a Docker image has been built and tagged with that SHA, and pushed to a Docker registry.

The alternative of simply using `latest` would cause several issues. For example, performing a rollback from `latest` to `latest` would not work. In addition, using the SHA means that you always know exactly which version of the code is build into the container.

However, as I also mentioned in that blog post, I'm not too happy with having to use `sed` to replace the version into the deployment file. To me this is a sign that I'm either doing something that I'm not supposed to do, or that a feature is missing. As I'm following a best practice of not using the `latest` tag, I'm pretty sure that this is an example of the latter situation.

## Helm

[Helm](https://helm.sh) is a package manager for Kubernetes. It can deploy multiple Kubernetes files and resources as a single package with a single lifecycle. This can simply be a set of resources distributed to different files (e.g. `deployment.yaml`, `service.yaml` etc.) or potentially multiple micro-services that together form an application.

If you are not yet familiar with Helm and have not yet installed it, check out the [Helm documentation](https://helm.sh/docs/using_helm/) to install it and follow along with the examples.

One of the most powerful features of Helm is the [templating](https://helm.sh/docs/chart_template_guide/). This makes it much easier to create templates that can be used by different teams for different purposes. This is especially interesting for templates such as [mysql/stable](https://github.com/helm/charts/tree/master/stable/mysql). This Helm Chart can be used for many different purposes such as for a sandbox environment or for production usage. However, the templating can also be used for a few nice-to-have features for more simple deployments that are more homogeneous.

Let me show you how we can replace the previous `<VERSION>` string with something that can be used within Helm:

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  [...]
spec:
  template:
    spec:
      containers:
          image: sanderknape/go-hello-world:{{ .Values.image.tag }}
  [...]
```

The `.Values.image.tag` is then set as:

`helm install go-hello-world ./ --set image.tag=123abc`

By default, Helm reads a `values.yaml` file with default values for any variables. Another approach therefore is to use this file that contains the tag:

**values.yaml**
```yaml
image.tag: 123abc
```

Everything combined, a deployment through GitLab could then look like this:

```yaml
deploy:
  stage: deploy
  image: dtzar/helm-kubectl
  script:
    - kubectl config set-cluster k8s --server="${SERVER}"
    - kubectl config set clusters.k8s.certificate-authority-data ${CERTIFICATE_AUTHORITY_DATA}
    - kubectl config set-credentials gitlab --token="${USER_TOKEN}"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default
    - helm init --client-only
    - helm upgrade go-hello-world ./ --install --set image.tag=${CI_COMMIT_SHORT_SHA}
```

In my [previous blog post](https://sanderknape.com/2019/02/automated-deployments-kubernetes-gitlab/) we used `kubectl apply` to apply the changes to the cluster - this is now replaced with the Helm command. More importantly, the `sed` is replaced with a more readable `--set` which is part of the templating feature within Helm.

As this does not look like a big change, you may wonder if learning and using Helm is really worth it for minimal changes like these. In my opinion, it is. The templating language alone already makes it much easier to deploy your applications to multiple environments (development, staging, production). For example, you can specify a different `values.yaml` file that contains a different number of replicas for each environment. You can also use the variables for some re-used values such as label names and names for deployments / services / pods. This makes the templates more re-usable for other applications as well. In general, there well be less copy/pasting required and less duplicate code and values.

In addition, the Helm templating [makes it easier to push ConfigMap changes to pods](https://sanderknape.com/2019/03/kubernetes-helm-configmaps-changes-deployments/) as I explain in my previous blog post. It's small wins like this that make it viable to use Helm. Note that I'm not using a [Helm Chart Repository](https://github.com/helm/helm/blob/master/docs/chart_repository.md) but deploy directly from the Git repository. This way we are not adding additional complexity in maintaining this repository.

Check out my [GitLab repository](https://gitlab.com/s.knape88/go-hello-world-k8s-helm) for a full example on using Helm to push deployments to Kubernetes using GitLab!
