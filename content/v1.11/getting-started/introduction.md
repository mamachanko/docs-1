---
title: Crossplane Introduction
weight: 2
---

Crossplane connects your Kubernetes cluster to external,
non-Kubernetes resources, and allows platform teams to build custom Kubernetes
APIs to consume those resources.

Crossplane creates Kubernetes
[Custom Resource Definitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
(`CRDs`) to represent the external resources as native 
[Kubernetes objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/). 
As native Kubernetes objects, you can use standard commands like `kubectl create`
and `kubectl describe`. The full 
[Kubernetes API](https://kubernetes.io/docs/reference/using-api/) is available
for every Crossplane resource. 

Crossplane also acts as a
[Kubernetes Controller](https://kubernetes.io/docs/concepts/architecture/controller/)
to monitor the state of the external resources and provide state enforcement. If
something modifies or deletes a resource outside of Kubernetes, Crossplane reverses
the change or recreates the deleted resource.

{{<img src="/media/crossplane-intro-diagram.png" alt="Diagram showing a user communicating to Kubernetes. Crossplane connected to Kubernetes and Crossplane communicating with AWS, Azure and GCP" align="center">}}
With Crossplane installed in a Kubernetes cluster, users only communicate with
Kubernetes. Crossplane manages the communication to external resources like AWS,
Azure or Google Cloud.

Crossplane also allows the creation of custom Kubernetes APIs. Platform teams can
combine external resources and simplify or customize the APIs presented to the
platform consumers.

## Crossplane components overview
This table provides a summary of Crossplane components and their roles. 

{{< table "table table-hover table-sm">}}
| Component | Abbreviation | Scope | Summary |
| --- | --- | --- | ---- | 
| [Provider]({{<ref "#providers">}}) | | cluster | Creates new Kubernetes Custom Resource Definitions for an external service. |
| [ProviderConfig]({{<ref "#provider-configurations">}}) | `PC` | cluster | Applies settings for a _Provider_. |
| [Managed Resource]({{<ref "#managed-resources">}}) | `MR` | cluster | A provider resource created and managed by Crossplane inside the Kubernetes cluster. | 
| [Composition]({{<ref "#compositions">}}) |  | cluster | A template for creating multiple _managed resources_ at once. |
| [Composite Resources]({{<ref "#composite-resources" >}}) | `XR` | cluster | Uses a _Composition_ template to create multiple _managed resources_ as a single Kubernetes object. |
| [Composite Resource Definitions]({{<ref "#composite-resource-definitions" >}}) | `XRD` | cluster | Defines the API schema for _Composite Resources_ and _Claims_ |
| [Claims]({{<ref "#claims" >}}) | `XRC` | namespace | Like a _Composite Resource_, but namespace scoped. | 
{{< /table >}}

## The Crossplane Pod
When installed in a Kubernetes cluster Crossplane creates an initial set of
Custom Resource Definitions (`CRDs`) of the core Crossplane components. 

{{< expand "View the initial Crossplane CRDs" >}}
After installing Crossplane use `kubectl get crds` to view the Crossplane
installed CRDs.

```shell {copy-lines="1"}
kubectl get crds
NAME                                                     
compositeresourcedefinitions.apiextensions.crossplane.io 
compositionrevisions.apiextensions.crossplane.io         
compositions.apiextensions.crossplane.io                 
configurationrevisions.pkg.crossplane.io                 
configurations.pkg.crossplane.io                         
controllerconfigs.pkg.crossplane.io                      
locks.pkg.crossplane.io                                  
providerrevisions.pkg.crossplane.io                      
providers.pkg.crossplane.io                              
storeconfigs.secrets.crossplane.io                       
```
{{< /expand >}}

The following sections describe the functions of some of these CRDs.

## Providers
A Crossplane _Provider_ creates a second set of CRDs that define how Crossplane
connects to a non-Kubernetes service. Each external service relies on its own
Provider. For example, 
[AWS](https://marketplace.upbound.io/providers/upbound/provider-aws), 
[Azure](https://marketplace.upbound.io/providers/upbound/provider-azure) 
and [GCP](https://marketplace.upbound.io/providers/upbound/provider-gcp)
are different providers for each cloud service.

{{< hint "tip" >}}
Most Providers are for cloud services but Crossplane can use a Provider to
connect to any service with an API.
{{< /hint >}}

For example, an AWS Provider installs Kubernetes CRDs for AWS resources like EC2
compute instances or S3 storage buckets.

The Provider defines the Kubernetes API definition for the external resource.
For example, the 
[Upbound Provider-AWS](https://marketplace.upbound.io/providers/upbound/provider-aws/)
defines a 
[`bucket`](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.25.0/resources/s3.aws.upbound.io/Bucket/v1beta1) 
resource for creating and managing AWS S3 storage buckets. 

Within the `bucket` CRD is a
[`spec.forProvider.region`](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.25.0/resources/s3.aws.upbound.io/Bucket/v1beta1#doc:spec-forProvider-region)
value that specifies which AWS region to deploy the bucket in.

The Upbound Marketplace contains a large 
[collection of Crossplane Providers](https://marketplace.upbound.io/providers).

More providers are available in the [Crossplane Contrib repository](https://github.com/crossplane-contrib/).

Providers are cluster scoped and available to all cluster namespaces.

View all installed Providers with the command `kubectl get providers`.

## Provider configurations
Providers have _ProviderConfigs_. _ProviderConfigs_ configure settings
related to the Provider like authentication or global defaults for the
Provider.

The API endpoints for ProviderConfigs are unique to each Provider.

_ProviderConfigs_ are cluster scoped and available to all cluster namespaces.

View all installed ProviderConfigs with the command `kubectl get providerconfig`.

## Managed Resources
A Provider's CRDs map to individual _resources_ inside the provider. When
Crossplane creates and monitors a resource it's a _Managed Resource_.

Using a Provider's CRD creates a unique _Managed Resource_. For example,
using the Provider AWS's `bucket` CRD, Crossplane creates a `bucket` _Managed Resource_
inside the Kubernetes cluster that's connected to an AWS S3 storage bucket.

The Crossplane controller provides state enforcement for _Managed Resources_.
Crossplane enforces the settings and existence of _Managed Resources_. This
"Controller Pattern" is like how the Kubernetes 
[kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
enforces state for pods.

_Managed Resources_ are cluster scoped and available to all cluster namespaces.

Use `kubectl get managed` to view all _managed resources_.
{{<hint "warning" >}}
The `kubectl get managed` creates a lot of Kubernetes API queries.
Both the `kubectl` client and kube-apiserver throttle the API queries. 

Depending on the size of the API server and number of managed resources, this
command may take minutes to return or may timeout. 

For more information, read 
[Kubernetes issue
#111880](https://github.com/kubernetes/kubernetes/issues/111880)
and 
[Crossplane issue #3459](https://github.com/crossplane/crossplane/issues/3459).
{{< /hint >}}

## Compositions

A _Composition_ is a template for a collection of _managed resources_.
_Compositions_ allow platform teams to specify a set of _managed resources_ as a
single object.

For example, a compute _managed resource_ may require the creation of a storage 
resource and a virtual network as well. A single _Composition_ can specify all
three resources, the compute, storage, and networking resources, in a single
_Composition_ object. 

Using _Compositions_ simplifies the deployment of infrastructure made up of
multiple _managed resources_. _Compositions_ also enforce standards and settings
across deployments.

Platform teams can specify fixed or default settings for each _managed resource_
inside a _Composition_ or specify fields and settings that users may change.

Using the previous example, the platform team may set a compute resource size
and virtual network settings. But the platform team only allows users to specify
the storage resource size.

Creating a _Composition_ doesn't create any managed resources. The _Composition_
is only a template for a collection of _managed resources_ and their settings. A
_Composite Resource_ creates the specific resources.

{{< hint "note" >}}
The _[Composite Resources]({{<ref "#composite-resources">}})_ section discusses
_Composite Resources_.
{{< /hint >}}

_Compositions_ are cluster scoped and available to all cluster namespaces.

Use `kubectl get compositions` to view all _compositions_.
 

 ## Composite Resources

A _Composite Resource_ (`XR`) is a set of provisioned _managed resources_. A
_Composite Resource_ uses the template specified by a _Composition_ and applies
any user specified settings. 

Multiple unique _Composite Resource_ objects can use the same _Composition_. For
example, a _Composition_ template can create a compute, storage and networking
set of _managed resources_.

If a _Composite Resource Definition_ (`XRD`) allows a user to specify resource
settings, users apply them in a _Composite Resource_ (`XR`).

{{< hint "tip" >}}
_Compositions_ are templates for a set of _managed resources_.  
_Composite Resources_ fill out the template and create _managed resources_.

Deleting a _Composite Resource_ deletes all the _managed resources_ it created.
{{< /hint >}}

_Composite Resources_ are cluster scoped and available to all cluster namespaces.

Use `kubectl get composite` to view all _Composite Resources_.

## Composite Resource Definitions
_Composite Resource Definitions_ (`XRDs`) create custom Kubernetes APIs used by 
_Claims_ and _Composite Resources_.

{{< hint "note" >}}
The _[Claims]({{<ref "#claims">}})_ section discusses
_Claims_.
{{< /hint >}}

Platform teams define the custom APIs.  
These APIs can specify specific values like storage space in gigabytes, generic
settings like `small` or `large`, or deployment options like `cloud` or
`onprem`. Crossplane doesn't limit the API definitions.

The _Composite Resource Definition's_ `kind` is from Crossplane.
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
```

The `spec` of a _Composite Resource Definition_ creates the  `apiVersion`,
`kind` and `spec` of a _Composite Resource_. 

{{< hint "tip" >}}
The _Composite Resource Definition_ defines the parameters for a _Composite
Resource_.
{{< /hint >}}

A _Composite Resource Definition_ has four main `spec` parameters:
* A {{<hover label="specGroup" line="3" >}}group{{< /hover >}} 
to define the 
{{< hover label="xr2" line="2" >}}apiVersion{{</hover >}} 
in a _Composite Resource_ .
* The {{< hover label="specGroup" line="7" >}}versions.name{{</hover >}} 
that defines the version used in a _Composite Resource_.
* A {{< hover label="specGroup" line="5" >}}names.kind{{</hover >}}
to define the _Composite Resource_ 
{{< hover label="xr2" line="3" >}}kind{{</hover>}}.
* A {{< hover label="specGroup" line="8" >}}versions.schema{{</hover>}} section
to define the _Composite Resource_ {{<hover label="xr2" line="6" >}}spec{{</hover >}}.

```yaml {label="specGroup"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: MyComputeResource
  versions:
  - name: v1alpha1
    schema:
      # Removed for brevity
```

A _Composite Resource_ based on this _Composite Resource Definition_ looks like this:

```yaml {label="xr2"}
# Composite Resource (XR)
apiVersion: test.example.org/v1alpha1
kind: MyComputeResource
metadata:
  name: my-resource
spec:
  storage: "large"
```

A _Composite Resource Definition_ {{< hover label="specGroup" line="8" >}}schema{{</hover >}} defines the _Composite Resource_
{{<hover label="xr2" line="6" >}}spec{{</hover >}} parameters.

These parameters are the new, custom APIs, that developers can use. 

For example, creating a compute _managed resource_ requires knowledge of a
cloud provider's compute class names like AWS's `m6in.large` or GCP's
`e2-standard-2`. 

A _Composite Resource Definition_ can limit the choices to `small` or `large`.
A _Composite Resource_ uses those options and the _Composition_ maps them
to specific cloud provider settings settings. 

The following _Composite Resource Definition_ defines a {{<hover label="specVersions" line="17" >}}storage{{< /hover >}}
parameter. The size is a 
{{<hover label="specVersions" line="18">}}string{{< /hover >}} 
and the OpenAPI 
{{<hover label="specVersions" line="19" >}}oneOf{{< /hover >}} requires the
options to be either {{<hover label="specVersions" line="20" >}}small{{< /hover >}} 
or {{<hover label="specVersions" line="21" >}}large{{< /hover >}}.

```yaml {label="specVersions"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: MyComputeResource
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              storage:
                type: string
                oneOf:
                  - pattern: '^small$'
                  - pattern: '^large$'
            required:
            - size  
```

A _Composite Resource Definition_ can define a wide variety of settings and
options. 

Creating a _Composite Resource Definition_ enables the creation of _Composite
Resources_. It can also enable the creation of _Claims_.

_Composite Resource Definitions_ with a `spec.claimNames` allow developers to
create _Claims_.

For example, the 
{{< hover label="xrdClaim" line="6" >}}claimNames.kind{{</hover >}}
allows the creation of _Claims_ of `kind: ComputeClaim`.
```yaml {label="xrdClaim"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: MyComputeResource
  claimNames:
    kind: ComputeClaim
  # Removed for brevity 
```

## Claims
_Claims_ are the primary way developers interact with Crossplane. 

_Claims_ access the custom APIs defined by the platform team in a _Composite
Resource Definition_.

_Claims_ look like _Composite Resources_, but they're namespace scoped,
while _Composite Resources_ are cluster scoped. 

{{< hint "note" >}}
**Why does namespace scope matter?**  
Having namespace scoped _Claims_ allows multiple teams, using unique namespaces,
to create the same types of resources, independent of each other. The compute
resources of team-A are unique to the compute resources of team-B.

Directly creating _Composite Resources_ requires cluster-wide permissions,
shared with all teams.   
_Claims_ create the same set of resources, but from a namespace level.
{{< /hint >}}

The previous _Composite Resource Definition_ allows the creation of _Claims_
of the kind  
{{<hover label="xrdClaim2" line="7" >}}ComputeClaim{{</hover>}}.  

Claims use the same 
{{< hover label="xrdClaim2" line="3" >}}apiVersion{{< /hover >}}
defined in _Composite Resource Definition_ and also used by 
_Composite Resources_.
```yaml {label="xrdClaim2"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: MyComputeResource
  claimNames:
    kind: ComputeClaim
  # Removed for brevity 
```

In an example _Claim_ the 
{{<hover label="claim" line="2">}}apiVersion{{< /hover >}}
matches the {{<hover label="xrdClaim2" line="3">}}group{{< /hover >}} in the
_Composite Resource Definition_. 

The _Claim_ {{<hover label="claim" line="3">}}kind{{< /hover >}} matches the
_Composite Resource Definition_ 
{{<hover label="xrdClaim2" line="7">}}claimNames.kind{{< /hover >}}.

```yaml {label="claim"}
# Claim
apiVersion: test.example.org/v1alpha1
kind: ComputeClaim
metadata:
  name: my-claim
  namespace: dev-group
spec:
  size: "large"
```

_Claims_ are namespace scoped. A Claim can specify a {{<hover label="claim"
line="6">}}namespace{{</hover >}}. A Claim that does not specify a namespace
will be created in the `default` namespace.
The _Composite Resource Definition_ defines the Claim's {{<hover label="claim"
line="7">}}spec{{< /hover >}} like a _Composite Resource_.

View all existing Claims with the command `kubectl get claim`.

## Next steps
Build your own Crossplane platform using one of the quickstart guides.
* [Azure Quickstart]({{<ref "provider-azure" >}})
* [AWS Quickstart]({{<ref "provider-aws" >}})
* [GCP Quickstart]({{<ref "provider-gcp" >}})