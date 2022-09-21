---
title: "KinD-a Neat"
date: 2022-09-21T18:13:36+12:00
draft: false
tags: ["terraform", "kubernetes", "kind", "helm"]
description: "☸ Kubernetes in Docker 🐳 cluster provisioning and configuration in Terraform (Helm and CRD's)"
showToc: true
TocOpen: true
---

I've recently become curious about the [OpenFaaS](https://www.openfaas.com/) framework,
having previously used [Lambda](https://aws.amazon.com/lambda/) and [Functions](https://github.com/Azure/Azure-Functions)
to act as 'event-driven glue' between cloud systems. I was also looking for something to dabble with in my spare time.
The prospect of learning a bit more about this new, but familiar, technology on top of my favourite container
orchestration system sounds grand.

More to come about learning that!

## 📦 Kubernetes Application Packages

That being said, I set out to install all the required components, many of which were provided through [Helm](https://helm.sh/).

Helm is great, but to install and maintain multiple deployments, a script containing various `add <x> repo`,
`deploy <y> chart` directives has to be added. Not to mention the various `values.yaml` files to maintain all the 
individual package configurations.

On top of that, you need to ensure that you have a valid `~/.kube/config` file, are in the right (non-production) cluster
as well as the right namespace.

Enter Terraform.

Terraform has a [Helm provider](https://registry.terraform.io/providers/hashicorp/helm) that allows you to declaratively
install a chart into Kubernetes. The [helm_release](https://registry.terraform.io/providers/hashicorp/helm/latest/docs/resources/release)
resource has many neat attributes such as the ability to `wait`, `create_namespace`, `set_sensitive` values, and specify
`atomic` rollbacks. 

The resultant configuration looks something like the following:

```terraform
resource "helm_release" "openfaas" {

  depends_on = [kubernetes_namespace.openfaas_core]

  name       = "faas"
  repository = "https://openfaas.github.io/faas-netes/"
  chart      = "openfaas"
  version    = "10.2.11"
  namespace  = kubernetes_namespace.openfaas_core.metadata[0].name

  set {
    name  = "functionNamespace"
    value = "openfaas-fn"
  }

  set {
    name  = "generateBasicAuth"
    value = "true"
  }
}
```

In a mere 20 lines we have managed to specify the upstream repository, chart name, version, and even set a few values.
It's a super nice and clean way of handling all configuration as code.

## 📄 Kubernetes Provider Config

I initially created te cluster via `KinD` manually, just to speed up the TTT (Time To Terraforming). Once the main installation
components were out of the way, I grappled with the provider configuration I had given Helm.

None.

Helm was happy to use the current context, current namespace of `kubectl` at apply time. While a safe enough thing to do,
mistakes happen, and fingers get tired of typing `kubectl config use-context <x>`.

A brief Google yielded the [kyma-incubator/kind](https://registry.terraform.io/providers/kyma-incubator/kind) provider,
which was perfect as it provides yet-another-means of reducing the amount of scripts and pre-installation steps I need to repeat.

It has the ability to configure `KinD` with `yaml`, as outlined in the upstream [KinD Configuration documentation](https://kind.sigs.k8s.io/docs/user/configuration/).
This means a new cluster could be provisioned repeatably and easily.

The resource provides [attributes](https://registry.terraform.io/providers/kyma-incubator/kind/latest/docs/resources/cluster#attributes-reference)
about the cluster it creates, specifically regarding authentication. Using interpolation, we can stack the outputs from the cluster
and use them in the Helm provider to ensure that we reference our local instance everytime.

This is wildly convenient, and [noted in the Kubernetes Terraform provider](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs#stacking-with-managed-kubernetes-cluster-resources),
along with a minor caveat (Entire warning copied to convey the full message):

> When using interpolation to pass credentials to the Kubernetes provider from other resources,
> these resources SHOULD NOT be created in the same Terraform module where Kubernetes provider resources are also used. 
> This will lead to intermittent and unpredictable errors which are hard to debug and diagnose. 
> The root issue lies with the order in which Terraform itself evaluates the provider blocks vs. actual resources. 
> Please refer to [this section](https://www.terraform.io/language/providers/configuration#provider-configuration)
> of Terraform docs for further explanation.
 
Sensible advice, but for our own local testing where the intention is to throw away the cluster and re-provision 
every other day, it's a risk I am willing to take. It's also been incredibly stable during development, albeit with limited
resources to plan.

Within the same module, the configuration looks like this:

```terraform
provider "helm" {
  kubernetes {
    host                   = kind_cluster.default.endpoint
    client_certificate     = kind_cluster.default.client_certificate
    client_key             = kind_cluster.default.client_key
    cluster_ca_certificate = kind_cluster.default.cluster_ca_certificate
  }
}
```

No room for ambiguity! 🎉

## ✍ Kubernetes Custom Resource Definitions (CRDs)

This is all very well and good, aside from the one small elephant in the room. CRD's.

Many Helm packages rely on new Kubernetes resources, Kubernetes needs to know about these before our Helm charts can install
the new custom "thing".

The [Helm documentation](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/) outlines that there is 
native support for CRD's within installable charts, however the CRD's will not be tracked, updated, or deleted. You can read
more about the design decisions at the CRD link, but most chart publishers will require you to apply the definition for these
resources prior to installing. 

Often there is a snippet in the installation docs that shows you a
`kubectl apply -f https://i-definitely-checked-this-file-contents-before-installing-in-prod.example.com` command, which
has always seems a little sketchy, and is reminiscent of `curl | sh` installation instructions that sysadmins are so fond of.

That being said, this is going to be installed on a local throw away cluster with no important data, so I'm game!

Hashicorp maintains a lot of providers, some of which serve as utility builtins. A full list can be seen 
[here](https://registry.terraform.io/search/providers?category=utility&namespace=hashicorp).

One of these is the [hashicorp/http](https://registry.terraform.io/providers/hashicorp/http/latest) module which provides a
`data` source allowing you to fetch a remote document.

This can easily be configured to retrieve the aforementioned sketchy link. Below is an example of the RabbitMQ CRD's.

```terraform
data "http" "cert_manager_crds" {
  url = "https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.crds.yaml"
}
```


```yml
theme: "PaperMod"
```

{{< highlight terraform "" >}}
data "http" "cert_manager_crds" {
  url = "https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.crds.yaml"
}
{{< / highlight >}}


From there, the `response_body` can be parsed and applied. 

### 💥 Applying Manifests

I found that the default [`kubectl_manifest`](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/manifest)
resource suffers from a slight downfall, the [issue](https://github.com/hashicorp/terraform-provider-kubernetes/issues/1380)
being one that [affects Terraform generally](https://github.com/hashicorp/terraform-provider-kubernetes/issues/1367).

Within a module, one can not install the CRD, as well as resources that depend on it. As the CRD is not installed on 
the server on the initial plan, Terraform fails to execute (The provider performs a Server Side Apply). 

A workaround as mentioned in first GitHub issue mentioned, is to use the [gavinbunney/kubectl](https://registry.terraform.io/providers/gavinbunney/kubectl/1.14.0)
provider, which performs a client side apply, and mitigates the downfall. The provider also provides a nice way to enumerate
resources based on a single yaml file, though I would have preferred to stick with one provider.

The end result looks like the following (One could download the CRD and open it with file instead of `data http` if 
they were security concious):

```terraform
# Retrieve the multi-resource YAML file from upstream
data "http" "cert_manager_crd" {
  url = "https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.crds.yaml"
}

# Parse the HTTP response body into individual manifests
data "kubectl_file_documents" "cert_manager_crd" {
  content = data.http.cert_manager_crd.response_body
}

# for_each manifest, apply the change
resource "kubectl_manifest" "cert_manager_crd" {
  //noinspection HILUnresolvedReference
  for_each  = data.kubectl_file_documents.cert_manager_crd.manifests
  yaml_body = each.value
}
```

Another benefit of the `gavinbunney` provider is that the manifest output map has a sensible key, resulting in a nice
non-numeric key within state

```shell
$ terraform state list
kubectl_manifest.cert_manager_crd["/apis/apiextensions.k8s.io/v1/customresourcedefinitions/certificaterequests.cert-manager.io"]
kubectl_manifest.cert_manager_crd["/apis/apiextensions.k8s.io/v1/customresourcedefinitions/certificates.cert-manager.io"]       
kubectl_manifest.cert_manager_crd["/apis/apiextensions.k8s.io/v1/customresourcedefinitions/challenges.acme.cert-manager.io"]    
kubectl_manifest.cert_manager_crd["/apis/apiextensions.k8s.io/v1/customresourcedefinitions/clusterissuers.cert-manager.io"]     
kubectl_manifest.cert_manager_crd["/apis/apiextensions.k8s.io/v1/customresourcedefinitions/issuers.cert-manager.io"]
kubectl_manifest.cert_manager_crd["/apis/apiextensions.k8s.io/v1/customresourcedefinitions/orders.acme.cert-manager.io"] 
```


## 🔚 Wrap-up

That's that! I really enjoy going down a bit of a rabbit hole to allow a stack to be setup with one command.
Terraform is a really strong provisioner of resources, despite some quirks that may rise along the way.

I'll throw more content up as I explore more neat stuff.
