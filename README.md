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

But if you have to do lot of Kubernetes resource modifications and validation depending on the Custom object state, it is recommended to use [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder).

## Building CRD using Kubebuilder

### Prerequisites
1. [Golang](https://go.dev/doc/install) is installed.
1. [Kubebuilder](https://book.kubebuilder.io/quick-start.html#installation) - version 3+ is installed.
1. [Minikube](https://minikube.sigs.k8s.io/docs/start/) is installed and running.

### What are we building ?
We are build a crd with name `SimpleService` for a simple web application. We are going to write a controller for CTS `SimpleService` which will create a deployment and service with name same as that of CRD. For below CRD object a deployment and service with name `simpleservice-sample` will be created.
```
apiVersion: com.myorg/v1
kind: SimpleService
metadata:
  name: simpleservice-sample
spec:
  image: nginx:1.14.2
  containerPort: 80
```
### Make a CRD
Open terminal inside `GOPATH` in a directory which matches your repo and execute below commands. If not under `GOPATH`, set the `--repo=<module path>` in below `kubebuilder init` command.
```
mkdir crd-sample && cd crd-sample
kubebuilder init --domain com.myorg

kubebuilder create api --version v1 --kind SimpleService
# Type "y" for the questions "Create Resource [y/n]" and "Create Controller [y/n]"
```

1. The file `./api/v1/simpleservice_types.go` defines the desired structure/schema of CRD. Update the `SimpleServiceSpec` and `SimpleServiceStatus` as given below.
```
type SimpleServiceSpec struct {
	Image            string `json:"image,omitempty"`
	ContainerPort    *int64 `json:"containerPort,omitempty"`
	Host             string `json:"host,omitempty"`
	IngressClassName string `json:"ingressClassName,omitempty"`
}

type SimpleServiceStatus struct {
	Status  string `json:"status,omitempty"`
	Message string `json:"message,omitempty"`
}
```


2. The file `./controllers/simpleservice_controller.go` contains the controller logic. It’s a controller’s job to ensure that, for any given object, the actual state of the world (both the cluster state, and potentially external state like running containers for Kubelet or loadbalancers for a cloud provider) matches the desired state in the object. In controller-runtime, the logic that implements the reconciling for a specific kind is called a Reconciler. A reconciler takes the name of an object, and returns whether or not we need to try again (e.g. in case of errors or periodic controllers, like the HorizontalPodAutoscaler).

Add Role changes to edit deployment and service

```
//+kubebuilder:rbac:groups=apps/v1,resources=deployment,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=v1,resources=service,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=com.myorg,resources=simpleservices,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=com.myorg,resources=simpleservices,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=com.myorg,resources=simpleservices/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=com.myorg,resources=simpleservices/finalizers,verbs=update
```


<details>

<summary>3. Add logic to "Reconcile" method in "./controllers/simpleservice_controller.go" file</summary>

```
func (r *SimpleServiceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)

	var simpleService commyorgv1.SimpleService
	if err := r.Client.Get(ctx, req.NamespacedName, &simpleService); err != nil {
		log.Error(err, "Unable to fetch SimpleService - ", req.NamespacedName.Namespace, req.NamespacedName.Name)
		return ctrl.Result{Requeue: false}, client.IgnoreNotFound(err)
	}

	defer func() {
		err := r.Client.Update(ctx, &simpleService)
		if err != nil {
			log.Error(err, "Error occurred while updating SimpleService, check status for detail")
		}
	}()

	if err := r.createOrUpdateDeployment(ctx, req, simpleService); err != nil {
		log.Error(err, "Error occurred while creating deployment")
		simpleService.Status.Status = "Failed"
		simpleService.Status.Message = "Failed to create/update deployment : " + err.Error()
		return ctrl.Result{Requeue: false}, err
	}

	if err := r.createOrUpdateService(ctx, req, simpleService); err != nil {
		log.Error(err, "Error occurred while creating service")
		simpleService.Status.Status = "Failed"
		simpleService.Status.Message = "Failed to create/update service : " + err.Error()
		return ctrl.Result{Requeue: false}, err
	}

	return ctrl.Result{Requeue: false}, nil
}

func (r *SimpleServiceReconciler) createOrUpdateService(ctx context.Context,
	req ctrl.Request, simpleService commyorgv1.SimpleService) error {
	var service core_v1.Service
	update := true
	var err error
	if err = r.Client.Get(ctx, req.NamespacedName, &service); err != nil {
		update = false
		service = core_v1.Service{
			ObjectMeta: metav1.ObjectMeta{
				Name:      simpleService.ObjectMeta.Name,
				Namespace: simpleService.ObjectMeta.Namespace,
				Labels: map[string]string{
					"app":     "SimpleService",
					"service": simpleService.ObjectMeta.Name,
				},
				OwnerReferences: r.buildOwnerReferences(simpleService),
			},
			Spec: core_v1.ServiceSpec{},
		}
	}

	service.Spec.Ports = []core_v1.ServicePort{
		{
			Name:     "http",
			Protocol: core_v1.ProtocolTCP,
			Port:     simpleService.Spec.ContainerPort,
		},
	}
	service.Spec.Selector = map[string]string{
		"app":     "SimpleService",
		"service": req.NamespacedName.Name,
	}
	if update {
		err = r.Client.Update(ctx, &service)
	} else {
		err = r.Client.Create(ctx, &service)
	}
	return err
}

func (r *SimpleServiceReconciler) buildOwnerReferences(simpleService commyorgv1.SimpleService) []metav1.OwnerReference {
	truePtr := true
	return []metav1.OwnerReference{
		{

			APIVersion:         commyorgv1.GroupVersion.String(),
			Kind:               "SimpleService",
			Name:               simpleService.ObjectMeta.Name,
			BlockOwnerDeletion: &truePtr,
			Controller:         &truePtr,
			UID:                simpleService.ObjectMeta.UID,
		},
	}
}

func (r *SimpleServiceReconciler) createOrUpdateDeployment(ctx context.Context,
	req ctrl.Request, simpleService commyorgv1.SimpleService) error {

	var deployment appsv1.Deployment
	update := true
	var err error
	if err = r.Client.Get(ctx, req.NamespacedName, &deployment); err != nil {
		update = false
		deployment = appsv1.Deployment{
			ObjectMeta: metav1.ObjectMeta{
				Name:      simpleService.ObjectMeta.Name,
				Namespace: simpleService.ObjectMeta.Namespace,
				Labels: map[string]string{
					"app":     "SimpleService",
					"service": simpleService.ObjectMeta.Name,
				},
				OwnerReferences: r.buildOwnerReferences(simpleService),
			},
			Spec: appsv1.DeploymentSpec{
				Replicas: r.int32Ptr(1),
				Selector: &metav1.LabelSelector{
					MatchLabels: map[string]string{
						"app":     "SimpleService",
						"service": req.NamespacedName.Name,
					},
				},
				Template: apiv1.PodTemplateSpec{
					ObjectMeta: metav1.ObjectMeta{
						Labels: map[string]string{
							"app":     "SimpleService",
							"service": req.NamespacedName.Name,
						},
					},
					Spec: apiv1.PodSpec{
						Containers: []apiv1.Container{
							{
								Name:  "web",
								Image: simpleService.Spec.Image,
								Ports: []apiv1.ContainerPort{
									{
										Name:          "http",
										Protocol:      apiv1.ProtocolTCP,
										ContainerPort: simpleService.Spec.ContainerPort,
									},
								},
							},
						},
					},
				},
			},
		}
	}

	deployment.Spec.Template.Spec.Containers[0].Image = simpleService.Spec.Image
	deployment.Spec.Template.Spec.Containers[0].Ports[0].ContainerPort = simpleService.Spec.ContainerPort
	if update {
		err = r.Client.Update(ctx, &deployment)
	} else {
		err = r.Client.Create(ctx, &deployment)
	}
	return err
}

func (r *SimpleServiceReconciler) int32Ptr(i int32) *int32 { return &i }
```

</details>

4. Make sure that "./go.mod" updated with latest version of library

```
k8s.io/api v0.23.6
k8s.io/apimachinery v0.23.6
k8s.io/client-go v0.23.6
```

### Deploy and Run Local
Make sure that the local kubectl config file is pointing to minikube. Run the below commands to deploy CRD.
```
# Update manifest whenever you make any changes to the API definitions or RBAC marker
make manifests
# Install the CRDs into the cluster:
make install

# Run your controller locally - make sure that the server starts
make run
```

### Test
Update the `config/samples/_v1_simpleservice.yaml` file with below content.
```
apiVersion: com.myorg/v1
kind: SimpleService
metadata:
  name: simpleservice-sample
spec:
  image: nginx:1.14.2
  containerPort: 80
```
**Test Create**
```
% kubectl apply -f config/samples/_v1_simpleservice.yaml
simpleservice.com.myorg/simpleservice-sample created
% kubectl get simpleservices.com.myorg 
NAME                   AGE
simpleservice-sample   17s
% kubectl get deployments.apps simpleservice-sample
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
simpleservice-sample   0/1     1            0           23s
% kubectl get service simpleservice-sample
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
simpleservice-sample   ClusterIP   10.103.214.222   <none>        80/TCP    33s
```

**Test Delete**
```
% kubectl delete simpleservices.com.myorg simpleservice-sample
simpleservice.com.myorg "simpleservice-sample" deleted
% kubectl get deployments.apps simpleservice-sample
Error from server (NotFound): deployments.apps "simpleservice-sample" not found
% kubectl get svc simpleservice-sample            
Error from server (NotFound): services "simpleservice-sample" not found
```

### Install it in minikube or remote cluster
Build and push your image to the location specified by IMG:
```
make docker-build docker-push IMG=<some-registry>/<project-name>:tag
```
Deploy the controller to the cluster with image specified by IMG:
```
make deploy IMG=<some-registry>/<project-name>:tag
```

## Reference

1. https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
1. https://book.kubebuilder.io/