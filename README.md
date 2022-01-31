# Sample Custom Resource Definition (CRD)

This project will give the basic idea on 
1. [What is CRD](#what-is-crd)
1. [Benefits of using CRD](#benefits-of-using-crd)
1. [Creating a CRD](#creating-a-crd)
1. [Creating a Custom object](#creating-a-custom-object)
1. [Watching the changes on Custom objects](#watching-the-changes-on-custom-objects)
1. [Building CRD using Kubebuilder](#building-crd-using-kubebuilder)

## Prerequisites
1. Basic understanding of Kubernetes.
1. Minikube running in local machine.
1. Kubectl must be installed locally and must be configured to hit the Minikube.
1. Golang must be installed locally and must have fair development knowledge for building CRD using Kubebuilder.

## What is CRD

In June 2019, with the release of Kubernetes version 1.15, the Custom Resource Definitions (CRDs) is introduced, which lets the developers to extend the capabilities of Kubernetes with custom models and business logic. What is more; accessing and managing these new resources is no different from regular Kubernetes resources. All the CLI (kubectl) and API that work for built-in Kubernetes resources will work for custom resources, with no additional effort from developers.

## Benefits of using CRD

### CRDs follow K8s Specification.

Since custom resources are just another type of Kubernetes resource, it comes with following benefits out of the box.

1. Kubernetes way of identifying resources with Group, Version and Kind applies to CRDs.
1. No need to perform authentication/authorization. The Kube API server can take care of it. The authorization can be configured through roles and rolebindings similar to other Kubernetes resources.
1. No need of separate api (deployment, services and ingress, and DNS entries). The Kube API server becomes gateway to your resources.
1. Automatically integrates to kubectl or any third-party system that can talk to kube-api-server

### Business Value of CRD

1. When providing self-managed Kubernetes clusters, often we come across a situation where we do not want to give users escalated permissions to create certain resources. For example; Let us say we want to let application teams to create namespaces for their applications. However we want to enforce certain constraints that they should follow proper naming convention, and that each application teams can only create “n” number of namespaces as specified by external configuration. We could create a “MyNamespace” CRD, and implement all this business logic in it. Now, application teams can be given authorization to create “MyNamespace”, but they will not be allowed to create Kubernetes namespace directly.
1. In an enterprise that has self-managed Kubernetes clusters, it is not uncommon to see Kubernetes resources are created and assigned to teams as per organization units (such as Inventory/Order/Sales), and functional units (such as dev/test/prod). We are going to need some kind of glue to link them together, mostly it requires external DB, and a microservice that abstracts it. With CRD, we could create “MyApplication”, “MyCluster”, or “MyEnvironment” custom resources, and they all can be maintained like a Kubernetes resource as discussed above. Cutting down cost of developing and maintaining external DB and an additional service that again has to be developed with security, scale, and performance requirements.

## Creating a CRD

First register a CRD in Kubernetes cluster. This is a way how telling the Kubernetes cluster the schema for our CRD, so that kubernetes api can support the Create/Update/Delete of crd object(`kind: DemoCrd` in our case).

CRD is registered by creating a object of kind `CustomResourceDefinition`. Use below commands to create our `democrds`.
```
git clone git@github.com:dinumathai/crd-sample.git
cd crd-sample
kubectl apply -f deploy/crd.yaml
```
Now we will check the content of [deploy/crd.yaml](deploy/crd.yaml)
1. `metadata > name` - Is the fully qualified name of the CRD.
1. `spec > group` and `spec > names` - Is used to specify CRD name and shortNames
1. `spec > scope` -  The CRD can be either  `Namespaced` or `Cluster` scoped. If scope is `namespaced`, the Custom objects is linked to a namespace and deleting a namespace deletes all Custom objects in that namespace.
1. `spec > versions > schema` - Schema of the Custom resource for the version specified in `name`. With OpenAPI v3.0 format validation a schema can be specified, which is validated during creation and updates. [read more](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema)
1. `conversion > webhook` - Instruct API server to call an external webhook for any conversion between two versions of custom resources. [read more](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)

## Creating a Custom object

Now we will create and object of kind `DemoCrd`. Please check [deploy/test-crd.yaml](deploy/test-crd.yaml).

The custom resource object can be created just like any kubernetes object. Use below commands to create and verify the custom resource object.
```
% kubectl apply -f deploy/test-crd.yaml 
customresourcedefinition.apiextensions.k8s.io/democrds.example.com created

% kubectl get democrds.example.com -n default    
NAME            AGE
democrds-test   19s

% kubectl get democrds -n default   
NAME            AGE
democrds-test   23s

% kubectl get dc -n default
NAME            AGE
democrds-test   28s
```
## Watching the changes on Custom objects

One of the use-case of CRD is to user it as a metadata. In that case we don't need to take any action depending on that change of the Custom object.

But most of the use-case on create of a Custom object some other kubernetes resource must be changed or some external resource must be changed. This can be achieved using 
1. [Admission webhooks](https://github.com/dinumathai/admission-webhook-sample)
2. Kubernetes client will emit events when any kubernetes resource is created/updated/deleted. For example [golang client](https://pkg.go.dev/k8s.io/client-go/tools/watch#Until). There are also third party library which makes CRD watching easier.

But if you have to do lot of action/validation depending on the Custom object state, it is recommended to use [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder).

## Building CRD using Kubebuilder

Coming Soon !!

## Reference

1. https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
1. https://book.kubebuilder.io/