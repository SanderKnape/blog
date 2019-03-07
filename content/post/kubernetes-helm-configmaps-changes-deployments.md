+++
title = "Using Kubernetes Helm to push ConfigMap changes to your Deployments"
author = "Sander Knape"
date = 2019-03-07T08:47:02+02:00
draft = false
tags = ["kubernetes", "helm", "configmap"]
categories = []
+++

In recent years Kubernetes has quickly gained a lot of popularity and it currently has huge momentum. Adoption is rising while at the same time, new users find out the areas where Kubernetes is still lacking.

One such area is the lifecycle management of application configuration. The construct in Kubernetes to store such configuration is the [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/). These ConfigMaps can be referenced from Pods or Deployments and the values can be injected to the container using environment variables or files through volumes.

However, the lifecycle of these ConfigMaps is completely separated from the containers that uses it. What this means is that when configuration is updated, the Pod or Deployment that uses it is not triggered and the container will not pick up the latest changes. It is only through re-creating the containers that configuration updates would become available.

The only exception here is when [mounting configuration to volumes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#mounted-configmaps-are-updated-automatically). When a ConfigMap is updated, the files in the volume are "*eventually updated as well*". This introduces unreliable deployments though. The configuration that is updated "at some point" may break the application but as this is an out-of-bound process from the deployment, rollbacks are harder to implement. Also, it goes against the notion of immutable containers.

Interestingly enough, a [GitHub issue](https://github.com/kubernetes/kubernetes/issues/22368) has been open regarding this issue since 2016. Almost 2.5 years later there is still no built-in solution in Kubernetes.

# Potential solutions

Of course the Kubernetes community is creative enough to come up with solutions to solve this drawback. One such solution was [recently shared by The New Stack](https://thenewstack.io/solving-kubernetes-configuration-woes-with-a-custom-controller/), who have created a Controller that continually tracks changes to ConfigMaps. It keeps a list of which Deployments use what ConfigMaps, and ensure that pods are recreated when a used ConfigMap is changed. This project can be found in [GitHub](https://github.com/pusher/wave).

A downside of this solution is that the updates to the Pods are still happening out-of-bounds from the overall deployment. To decrease the load on the Kubernetes API server, by default the Controller polls for any changes every 5 minutes. It can therefore potentially take 5 minutes until Pods are being restarted. In a fully automated deployment pipeline where, for example, a deployment to staging is followed by an automated test, it would take some more complexity to account for this eventual consistency.

A simpler method is to change the name of the ConfigMap each time it is updated. Both the name of the ConfigMap and the references to the ConfigMap must then be updated. This gives you the most control as you have to make explicit changes that causes the pods to restart. In addition, if you use the ConfigMaps in multiple locations, you can update a single usage at a time to reduce the blast radius in case of an error.

While this definitely gives you the most control, its biggest downside is that this would cause a lot of manual work. You have to make sure that all references to ConfigMaps are updated when the name of a ConfigMap is changed. If you have setup a solid deployment process with proper health checks and rollbacks, erroneous changes to ConfigMaps could potentially already be cought in such a process without downtime. What is great though is that for changes with more risk, this method can be used while "typical" deployments can be automated through the other methods discussed in this post.

Another solution that is much more automatable is through the use of Pod [annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/). An annotation is basically an arbitrary key/value pair that can be used by tools to get or set additional information. What's interesting is that when a Deployment annotation is updated, the Deployment resource is considered "changed" which will trigger a recreation of the Pods. As is explained in [this blog post by Matt Silverlock](https://blog.questionable.services/article/kubernetes-deployments-configmap-change/), the value of such an annotation can be changed when the contents of a ConfigMap changes. This is done by generating a `sha256` hash of the ConfigMap, which of course will be different when the ConfigMap has changed.

In the example by Matt, a simple script is used that generates the hash and writes it to the pod annotation. Such a script can easily be added to an automated deployment pipeline. The main benefit is that restarting the Pods is now part of the overall deployment lifecycle. Pushing the new YAML with a different hash will immediatly restart the Pods. The deployment pipeline can watch the status of the new Pods and potentially roll back if there are any issues.

Things can be made a little simpler when using [Helm](https://helm.sh/). Let's look into this next.

# Jackpot?

The final solution we'll discuss - and the one that is the most feasible to me - is especially interesting when you are already using Helm. Documented as a [tip & trick](https://github.com/helm/helm/blob/master/docs/charts_tips_and_tricks.md#automatically-roll-deployments-when-configmaps-or-secrets-change) when using Helm, the method that I described previously can easily be done within Helm. Check out the following example script:

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

As part of the [Helm templating language](https://helm.sh/docs/chart_template_guide/) it is possible to create a `sha256` hash of a string. This, combined with the feature to load templates as strings, means we can create a one-liner that generates this hash for us right in the template.

The advantage compared to the previous solution is that no additional scripting is required. Now when reading the YAML template, the logic that is executed is clearly and explicetly visible. With the other method the `checksum/config` annotation would either be a placeholder or it would not exist at all. That is certainly more confusing to understand when reading the template for the first time.

# Conclusion

I'm suprised to see that a relatively simple process - a configuration change - is not yet fully standardized within the Kubernetes ecosystem. Considering that the ConfigMap is essentially a dependency for a Pod, it would make sense to me that a change of that ConfigMap could optionally trigger a restart of any Pods that are using it.

If you are already using Helm I would definitely recommend using the last method I described above. If you are not using Helm, you can easily implement the functionality yourself by generating and writing the hash yourself.

If you have a different way for handling ConfigMap changes, let me know in the comments!