# Working With Kustomize

In the [previous lab](../../README.md), you were able to deploy
some Kubernetes manifests and keep them in sync with your cluster
using ArgoCD. This is a powerful tool, but can lead to a lot of YAML
being copied everywhere. In a lot of cases, you'd like to deploy an
application to multiple places with only minor changes. How
do you do that without copying YAML everywhere? This is where
[`kustomize`](https://kustomize.io/) comes
in to save the day!

Before you get started, make sure you have the [ArgoCD Operator
installed](../../README.md#installing-the-argocd-operator) and an [ArgoCD
instance up](../../README.md#installing-an-argocd-instance) and running.

In this section we'll be going over:

* [Creating An App With Kustomize](#creating-an-app-with-kustomize)
* [Deploying An App With Kustomize](#deploying-an-app-with-kustomize)

If you haven't already, delete the previous application before starting.

```shell
oc delete -f resources/manifests/bgd-app
```

# Creating An App With Kustomize

In the previous lab, we [deployed a sample
application](../../README.md#deploying-a-sample-application), which
deployed the app with a blue square. But what if you wanted a green
square without copying all the configuration files? Use Kustomize!

If you look in [this repo](../manifests/bgdk-kustomize),
you'll see that I have a [kustomize
file](../manifests/bgdk-kustomize/kustomization.yaml). Let's go over this.

```yaml
resources:
- github.com/RedHatWorkshops/argocd-getting-started/resources/manifests/bgdk-yaml
patchesJson6902:
  - target:
      version: v1
      group: apps
      kind: Deployment
      name: bgd
      namespace: bgd
    path: bgd-deployment.yaml
```

As you'll see there's not much to it! Let's explore a bit.

* `resources` - This is how you load YAMLs. Notice that I can point to
another github repo. This saves me from having to copy the yaml.
* `patchesJson6902` - This tells Kustomize that I want to patch one of
the resources. In this case, the `Deployment`
* `patchesJson6902.target.path` - This tells Kustomize that I want to
use a file to specify what I'm patching.

> :bulb: Learn more about `patchesJson6902` and what other choices you have by visiting the [official documentation page](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patchesjson6902/)

Let's take a look at the [`bgd-deployment.yaml`](../manifests/bgdk-kustomize/bgd-deployment.yaml) file.

```yaml
- op: replace
  path: /spec/template/spec/containers/0/env/0/value
  value: green
```

> Not a lot either, right? Who says GitOps has to be hard? :smiley:

Although simple, let's break it down a bit.

* `op` - stands for operation. Here we are replacing a value.
* `path` - This is the path in the `Deployment`
manifest that you will replace. In this case the
[`color`](../manifests/bgdk-yaml/bgd-deployment.yaml#L26-L27) key's
value will be replaced with "green".

Having this in place, we're ready to deploy the application!

# Deploying An App With Kustomize

To deploy this application, you'll have to create an ArgoCD
[`Application`](../manifests/bgdk-appk/bgdk-appk.yaml).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgdk-appk
  namespace: openshift-gitops
spec:
  destination:
    namespace: openshift-gitops
    server: https://kubernetes.default.svc
  project: default
  source:
    path: resources/manifests/bgdk-kustomize
    repoURL: https://github.com/RedHatWorkshops/argocd-getting-started
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  sync:
    comparedTo:
      destination:
        namespace: openshift-gitops
        server: https://kubernetes.default.svc
      source:
        path: resources/manifests/bgdk-kustomize
        repoURL: https://github.com/RedHatWorkshops/argocd-getting-started
        targetRevision: main
```

This should look familiar if you went through the [previous
lab](../../README.md). Here you're specifying where to find the
YAML. But since Kustommize is built-in to ArgoCD, ArgoCD will see the
[`kustomize`](../manifests/bgdk-kustomize/kustomization.yaml) file and
perform a `kustomize build` before applying the manifest. Let's apply
this manifest.

```shell
oc apply -k resources/manifests/bgdk-appk
```

> :bulb: **NOTE**: You're actually using Kustomize to deploy the app that's using Kustomize! That's what the `-k` is in `oc apply`. Take a look at [the repo](../manifests/bgdk-appk) to to investigate further.

If you look at your ArgoCD WebUI, you should see the app fully synced
after a while.

![syncd-appk](../images/synced-appk.png)

Click on the card and look at the overview of your application.

![syncd-appk](../images/appk-overview.png)

You'll see that all the YAML manifest of the other repo was loaded
in. Cool! You can now see how you can deploy your one application in
many locations without copying tons of YAML everywhere!

Get the route of the application.

```shell
oc get route bgd -n bgd -o jsonpath='{.spec.host}{"\n"}'
```

Visiting the app, you should see a green square.

![green-square](../images/green-square.png)

Success :tada:

# Conclusion

You can now see how you can use Kustomize for deploying applications
accross many environments. Using Kustomize reduces the amount of YAML
needed to be copied to every environment by using templeting and patching.

To learn more, read the [official Kustomize documentation](https://kubernetes.io/docs/home/)
