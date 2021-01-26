# A simple Kubernetes Operator to return vSphere First Class Disk (FCD) / Improved Virtual Disk (IVD) information #

This repository contains a very simple Kubernetes Operator that uses VMware's __govmomi__ to return some simple First Class Disk / Imrpoved Virtual Disk (hereafter referred to as FCD) information through the status field of a __Custom Resource (CR)__, which is called ```FCDInfo```. This will require us to extend Kubernetes with a new __Custom Resource Definition (CRD)__. The code shown here is for education purposes only, showing one way in which a Kubernetes controller / operator can access the underlying vSphere infrastructure for the purposes of querying resources.

This is the third operator in this series. The previous operators ```HostInfo``` and ```VMInfo``` returned information related to ESXi hosts and vSphere Virtual Machines. This operator is focusing on getting further vSphere information from an FCD which is the vSphere storage object which backs a Kubernetes Persistent Volume (PV).

You can think of a CRD as representing the desired state of a Kubernetes object or Custom Resource, and the function of the operator is to run the logic or code to make that desired state happen - in other words the operator has the logic to do whatever is necessary to achieve the object's desired state.

## What are we going to do in this tutorial? ##

In this example, we will create a CRD called ```FCDInfo```. FCDInfo will contain the name of a Persistent Volume in its specification. When a Custom Resource (CR) is created and subsequently queried, we will call an operator (logic in a controller) whereby the size of the backing FCD (in MB), the path to the FCD and the format of the FCD will be returned via the status fields of the object through govmomi API calls.

The following will be created as part of this tutorial:

* A __Customer Resource Definition (CRD)__
  * Group: ```Topology```
    * Kind: ```FCDInfo```
    * Version: ```v1```
    * Specification will include a single item: ```Spec.pvId```

* One or more __FCDInfo Custom Resource / Object__ will be created through yaml manifests, each manifest containing the hostname of an ESXi host that we wish to query. The fields which will be updated to contain the relevant information from the ESXi host (when the CR is queried) are:
  * ```Status.filePath```
  * ```Status.provisioningType```
  * ```Status.sizeMB```

* An __Operator__ (or business logic) to retrieve the file path, provisioning type and size of the FCD specified in the CR will be coded in the controller for this CR.

__Note:__ As mentioned, there is a similar tutorial to create an operator to get both ESXi host information and virtual machine information. These can be found [here](https://github.com/cormachogan/hostinfo-operator) and [here](https://github.com/cormachogan/vminfo-operator).

## What is not covered in this tutorial? ##

The assumption is that you already have a working Kubernetes cluster. Installation and deployment of a Kubernetes is outside the scope of this tutorial. If you do not have a Kubernetes cluster available, consider using __Kubernetes in Docker__ (shortened to __Kind__) which uses containers as Kubernetes nodes. A quickstart guide can be found here:

* [Kind (Kubernetes in Docker)](https://kind.sigs.K8s.io/docs/user/quick-start/)

The assumption is that you also have a __VMware vSphere environment__ comprising of at least one ESXi hypervisor which is managed by a vCenter server. While the thought process is that your Kubernetes cluster will be running on vSphere infrastructure, and thus this operator will help you examine how the underlying vSphere resources are being consumed by the Kubernetes clusters running on top, it is not necessary for this to be the case for the purposes of this tutorial. You can use this code to query any vSphere environment from Kubernetes.

Lastly, there is an expectation that at least one Persistent Volume has been created on a Kubernetes cluster which uses the vSphere CSI driver. This will create an FCD on vSphere to back the PV.

## What if I just want to understand some basic CRD concepts? ##

If this sounds even too daunting at this stage, I strongly recommend checking out the excellent tutorial on CRDs from my colleague, __Rafael Brito__. His [RockBand](https://github.com/brito-rafa/k8s-webhooks/blob/master/single-gvk/README.md) CRD tutorial uses some very simple concepts to explain how CRDs, CRs, Operators, spec and status fields work, and is a great place to get started on your operator journey.

## Step 1 - Software Requirements ##

You will need the following components pre-installed on your desktop or workstation before we can build the CRD and operator.

* A __git__ client/command line
* [Go (v1.15+)](https://golang.org/dl/) - earlier versions may work but I used v1.15.
* [Docker Desktop](https://www.docker.com/products/docker-desktop)
* [Kubebuilder](https://go.kubebuilder.io/quick-start.html)
* [Kustomize](https://kubernetes-sigs.github.io/kustomize/installation/)
* Access to a Container Image Repositor (docker.io, quay.io, harbor)
* A __make__ binary - used by Kubebuilder

If you are interested in learning more about Golang basics, I found [this site](https://tour.golang.org/welcome/1) very helpful.

## Step 2 - KubeBuilder Scaffolding ##

The CRD is built using [kubebuilder](https://go.kubebuilder.io/).  I'm not going to spend a great deal of time talking about __KubeBuilder__. Suffice to say that KubeBuilder builds a directory structure containing all of the templates (or scaffolding) necessary for the creation of CRDs. Once this scaffolding is in place, this turorial will show you how to add your own specification fields and status fields, as well as how to add your own operator logic. In this example, our logic will login to vSphere, query and return FCD information  via a Kubernetes CR / object / Kind called HostInfo, the values of which will be used to populate status fields in our CRs.

The following steps will create the scaffolding to get started.

```cmd
mkdir fcdinfo
$ cd fcdinfo
```

Next, define the Go module name of your CRD. In my case, I have called it __fcdinfo__. This creates a __go.mod__ file with the name of the module and the Go version (v1.15 here).

```cmd
$ go mod init fcdinfo
go: creating new go.mod: module fcdinfo
```

```cmd
$ ls
go.mod
```

```cmd
$ cat go.mod
module fcdinfo

go 1.15
```

Now we can proceed with building out the rest of the directory structure. The following __kubebuilder__ commands (__init__ and __create api__) creates all the scaffolding necessary to build our CRD and operator. You may choose an alternate __domain__ here if you wish. Simply make note of it as you will be referring to it later in the tutorial.

```cmd
kubebuilder init --domain corinternal.com
```

Here is what the output from the command looks like:

```cmd
$ kubebuilder init --domain corinternal.com
Writing scaffold for you to edit...
Get controller runtime:
$ go get sigs.k8s.io/controller-runtime@v0.5.0
Update go.mod:
$ go mod tidy
Running make:
$ make
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/usr/share/go/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go
Next: define a resource with:
$ kubebuilder create api
$
```

As the output from the previous command states, we must now define a resource. To do that, we again use kubebuilder to create the resource, specifying the API group, its version and supported kind. My group is called topology, my kind is called FCDInfo and my initial version is v1.

```cmd
kubebuilder create api \
--group topology       \
--version v1           \
--kind FCDInfo         \
--resource=true        \
--controller=true
```

Here is the output from that command. Note that it is building the __types.go__ and __controller.go__, both of which we will be editing shortly:

```cmd
$ kubebuilder create api --group topology --version v1 --kind FCDInfo --resource=true --controller=true
Writing scaffold for you to edit...
api/v1/fcdinfo_types.go
controllers/fcdinfo_controller.go
Running make:
$ make
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/usr/share/go/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go
```

Our operator scaffolding (directory structure) is now in place. The next step is to define the specification and status fields in our CRD. After that, we create the controller logic which will watch our Custom Resources, and bring them to desired state (called a reconcile operation). More on this shortly.

## Step 3 - Create the CRD ##

Customer Resource Definitions [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) are a way to extend Kubernetes through Custom Resources. We are going to extend a Kubernetes cluster with a new custom resource called __FCDInfo__ which will retrieve information from a PV whose name is specified in a Custom Resource. Thus, I will need to create a field called __PVId__ in the CRD - this defines the specification of the custom resource. We also add three status fields, as these will be used to return information about the First Class Disk (FCD) backing the PV.

This is done by modifying the __api/v1/fcdinfo_types.go__ file. Here is the initial scaffolding / template provided by kubebuilder:

```go
// FCDInfoSpec defines the desired state of FCDInfo
type FCDInfoSpec struct {
        // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
        // Important: Run "make" to regenerate code after modifying this file

        // Foo is an example field of FCDInfo. Edit HostInfo_types.go to remove/update
        Foo string `json:"foo,omitempty"`
}

// FCDInfoStatus defines the observed state of FCDInfo
type FCDInfoStatus struct {
        // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
        // Important: Run "make" to regenerate code after modifying this file
}

// +kubebuilder:object:root=true
```

This file is modified to include a single __spec.PVId__ field and to return three __status__ fields. There are also a number of kubebuilder fields added, which are used to do validation and other kubebuilder related functions. The shortname "__fcd__" will be used later on in our controller logic. This can also be used with kubectl, e.g ```kubectl get fcd``` rather than```kubectl get fcdinfo```. Also, when we query any Custom Resources created with the CRD, e.g. ```kubectl get fcdinfo```, we want the output to display the PVId.

Note that what we are doing here is for education purposes only. Typically what you would observe is that the spec and status fields would be similar, and it is the function of the controller to reconcile and differences between the two to achieve eventual consistency. But we are keeping things simple, as the purpose here is to show how vSphere can be queried from a Kubernetes Operator. Below is a snippet of the __fcdinfo_types.go__ showing the code changes. It does not include the __imports__ which also need to be added.  The code-complete [fcdinfo_types.go](api/v1/fcdinfo_types.go) is here.

```go
// FCDInfoSpec defines the desired state of FCDInfo
type FCDInfoSpec struct {
        PVId string `json:"pvId"`
}

// FCDInfoStatus defines the observed state of FCDInfo
type FCDInfoStatus struct {
        SizeMB           int64  `json:"sizeMB"`
        FilePath         string `json:"filePath"`
        ProvisioningType string `json:"provisioningType"`
}

// +kubebuilder:validation:Optional
// +kubebuilder:resource:shortName={"fcd"}
// +kubebuilder:printcolumn:name="PVId",type=string,JSONPath=`.spec.pvId`
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
```

We are now ready to create the CRD. There is one final step however, and this involves updating the __Makefile__ which kubebuilder has created for us. In the default Makefile created by kubebuilder, the following __CRD_OPTIONS__ line appears:

```Makefile
# Produce CRDs that work back to Kubernetes 1.11 (no version conversion)
CRD_OPTIONS ?= "crd:trivialVersions=true"
```

This CRD_OPTIONS entry should be changed to the following:

```Makefile
# Produce CRDs that work back to Kubernetes 1.11 (no version conversion)
CRD_OPTIONS ?= "crd:preserveUnknownFields=false,crdVersions=v1,trivialVersions=true"
```

Now we can build our CRD with the spec and status fields that we have place in the __api/v1/fcdinfo_types.go__ file.

```cmd
make manifests && make generate
```

Here is the output from the make:

```Makefile
$ make manifests && make generate
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/usr/share/go/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/usr/share/go/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
```

## Step 4 - Install the CRD ##

The CRD is not currently installed in the Kubernetes Cluster.

```shell
$ kubectl get crd
NAME                                                               CREATED AT
antreaagentinfos.clusterinformation.antrea.tanzu.vmware.com        2020-11-18T17:14:03Z
antreacontrollerinfos.clusterinformation.antrea.tanzu.vmware.com   2020-11-18T17:14:03Z
clusternetworkpolicies.security.antrea.tanzu.vmware.com            2020-11-18T17:14:03Z
traceflows.ops.antrea.tanzu.vmware.com                             2020-11-18T17:14:03Z
```

To install the CRD, run the following make command:

```cmd
make install
```

The output should look something like this:

```makefile
$ make install
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/usr/share/go/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/fcdinfoes.topology.corinternal.com created
```

Now check to see if the CRD is installed running the same command as before.

```shell
$ kubectl get crd
NAME                                                               CREATED AT
antreaagentinfos.clusterinformation.antrea.tanzu.vmware.com        2020-11-18T17:14:03Z
antreacontrollerinfos.clusterinformation.antrea.tanzu.vmware.com   2020-11-18T17:14:03Z
clusternetworkpolicies.security.antrea.tanzu.vmware.com            2020-11-18T17:14:03Z
fcdinfoes.topology.corinternal.com                                 2021-01-25T13:40:41Z
traceflows.ops.antrea.tanzu.vmware.com                             2020-11-18T17:14:03Z
```

Our new CRD ```fcdinfoes.topology.corinternal.com``` is now visible. Another useful way to check if the CRD has successfully deployed is to use the following command against our API group. Remember back in step 2 we specified the domain as ```corinternal.com``` and the group as ```topology```. Thus the command to query api-resources for this CRD is as follows:

```shell
$ kubectl api-resources --api-group=topology.corinternal.com
NAME         SHORTNAMES   APIGROUP                   NAMESPACED   KIND
fcdinfoes    fcd           topology.corinternal.com   true        FCDInfo
```

## Step 5 - Test the CRD ##

At this point, we can do a quick test to see if our CRD is in fact working. To do that, we can create a manifest file with a Custom Resource that uses our CRD, and see if we can instantiate such an object (or custom resource) on our Kubernetes cluster. Fortunately kubebuilder provides us with a sample manifest that we can use for this. It can be found in __config/samples__.

```shell
$ cd config/samples
$ ls
topology_v1_fcdinfo.yaml
```

```yaml
$ cat topology_v1_fcdinfo.yaml
apiVersion: topology.corinternal.com/v1
kind: FCDInfo
metadata:
  name: fcdinfo-sample
spec:
  # Add fields here
  foo: bar
```

We need to slightly modify this sample manifest so that the specification field matches what we added to our CRD. Note the spec: above where it states 'Add fields here'. We have removed the __foo__ field and added a __spec.pvId__ field, as per the __api/v1/fcdinfo_types.go__ modification earlier. Thus, after a simple modification, the CR manifest looks like this, where ____ is the name of the ESXi host that we wish to query.

```yaml
$ cat topology_v1_fcdinfo.yaml
apiVersion: topology.corinternal.com/v1
kind: FCDInfo
metadata:
  name: fcdinfo-sample
spec:
  # Add fields here
  pvId:  pvc-e3f6dd59-cbc0-49a7-97c8-d92a26732c43
```

To see if it works, we need to create this HostInfo Custom Resource.

```shell
$ kubectl create -f topology_v1_fcdinfo.yaml
fcdinfo.topology.corinternal.com/fcdinfo-sample created
```

```shell
$ kubectl get fcdinfo
NAME              PVID
fcdinfo-sample    pvc-e3f6dd59-cbc0-49a7-97c8-d92a26732c43
```

Or use the shortcut, "fcd":

```shell
$ kubectl get fcd
NAME              PVID
fcdinfo-sample    pvc-e3f6dd59-cbc0-49a7-97c8-d92a26732c43
```

Note that the PVID field is also printed, as per the kubebuilder directive that we placed in the __api/v1/fcdinfo_types.go__. As a final test, we will display the CR in yaml format.

```yaml
$ kubectl get fcd fcdinfo -o yaml
apiVersion: topology.corinternal.com/v1
kind: FCDInfo
metadata:
  creationTimestamp: "2021-01-25T16:18:20Z"
  generation: 1
  managedFields:
  - apiVersion: topology.corinternal.com/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        .: {}
        f:pvId: {}
    manager: kubectl
    operation: Update
    time: "2021-01-25T16:18:20Z"
  name: fcdinfo-sample
  namespace: default
  resourceVersion: "32405581"
  selfLink: /apis/topology.corinternal.com/v1/namespaces/default/fcdinfoes/fcdinfo-sample
  uid: 56bd9a46-2b4d-4ace-843a-38a48b21b547
spec:
  pvId: pvc-e3f6dd59-cbc0-49a7-97c8-d92a26732c43
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

## Step 6 - Create the controller / manager ##

This appears to be working as expected. However there are no __Status__ fields displayed with our PV information in the __yaml__ output above. To see this information, we need to implement our operator / controller logic to do this. The controller implements the desired business logic. In this controller, we first read the vCenter server credentials from a Kubernetes secret (which we will create shortly). We will then open a session to my vCenter server, and get a list of First Class Disks that it manages. I then look for the FCD that is specified in the spec.pvId field in the CR, and retrieve the information this its backing FCD. Finally we will update the appropriate Status field with this information, and we should be able to query it using the __kubectl get fcdinfo -o yaml__ command seen previously.

### Step 6.1 - Open a session to vSphere ###

__Note:__ Let's first look at the login function which resides in __main.go__. This __vlogin__ function creates the vSphere session in main.go. One thing to note is that I am enabling insecure logins (true) by default. This is something that you may wish to change in your code. One other item to note is that I am testing two different client logins here, govmomi.Client and vim25.Client. The govmomi.Client uses Finder for getting vsphere information, and treats the vSphere inventory as a virtual filesystem. The vim25.Client uses ContainerView, and tends to generate more response data. As mentioned, this is a tutorial, so this operator shows both login types simply for imformational purposes.

```go

//
// - vSphere session login function
//

func vlogin(ctx context.Context, vc, user, pwd string) (*vim25.Client, *govmomi.Client, error) {

//
// This section allows for insecure govmomi logins
//

        var insecure bool
        flag.BoolVar(&insecure, "insecure", true, "ignore any vCenter TLS cert validation error")

//
// Create a vSphere/vCenter client
//
// The govmomi client requires a URL object, u.
// You cannot use a string representation of the vCenter URL.
// soap.ParseURL provides the correct object format.
//

        u, err := soap.ParseURL(vc)

        if u == nil {
                setupLog.Error(err, "Unable to parse URL. Are required environment variables set?", "controller", "VMInfo")
                os.Exit(1)
        }

        if err != nil {
                setupLog.Error(err, "URL parsing not successful", "controller", "VMInfo")
                os.Exit(1)
        }

        u.User = url.UserPassword(user, pwd)

//
// Session cache example taken from https://github.com/vmware/govmomi/blob/master/examples/examples.go
//
// Share govc's session cache
//
        s := &cache.Session{
                URL:      u,
                Insecure: true,
        }

//
// Create new vim25 client
//
        c1 := new(vim25.Client)

//
// Login using vim25 client c and cache session s
//
        err = s.Login(ctx, c1, nil)

        if err != nil {
                setupLog.Error(err, "FCDInfo: vim25 login not successful", "controller", "VMInfo")
                os.Exit(1)
        }

//
// Create new govmomi client
//

        c2, err := govmomi.NewClient(ctx, u, insecure)

        if err != nil {
                setupLog.Error(err, "FCDInfo: gomvomi login not successful", "controller", "VMInfo")
                os.Exit(1)
        }

        return c1, c2, nil
}
```

Within the main function, there is a call to the __vlogin__ function with the parameters received from the environment variables shown below. 

```go
//
// Retrieve vCenter URL, username and password from environment variables
// These are provided via the manager manifest when controller is deployed
//

        vc := os.Getenv("GOVMOMI_URL")
        user := os.Getenv("GOVMOMI_USERNAME")
        pwd := os.Getenv("GOVMOMI_PASSWORD")

//
// Create context, and get vSphere session information
//

        ctx, cancel := context.WithCancel(context.Background())
        defer cancel()

        c1, c2, err := vlogin(ctx, vc, user, pwd)
        if err != nil {
                setupLog.Error(err, "unable to get login session to vSphere")
                os.Exit(1)
        }

        finder := find.NewFinder(c2.Client, true)

//
// -- find and set the default datacenter
//

        dc, err := finder.DefaultDatacenter(ctx)

        if err != nil {
                setupLog.Error(err, "FCDInfo: Could not get default datacenter")
        } else {
                finder.SetDatacenter(dc)
        }
```

There is also an updated __FCDInfoReconciler__ call with new fields (VC1 & VC2) which have the vSphere session details. This login information can now be used from within the FCDInfoReconciler controller function, as we will see shortly.

```go
 if err = (&controllers.FCDInfoReconciler{
                Client: mgr.GetClient(),
                VC1:    c1,
                VC2:    c2,
                Finder: finder,
                Log:    ctrl.Log.WithName("controllers").WithName("FCDInfo"),
                Scheme: mgr.GetScheme(),
        }).SetupWithManager(mgr); err != nil {
                setupLog.Error(err, "unable to create controller", "controller", "FCDInfo")
                os.Exit(1)
```

Click here for the complete [__main.go__](./main.go) code.

### Step 6.2 - Controller Reconcile Logic ###

Now we turn our attention to the business logic of the controller. Once the business logic is added in the controller, it will need to be able to run in a Kubernetes cluster. To achieve this, a container image to run the controller logic must be built. This will be provisioned in the Kubernetes cluster using a Deployment manifest. The deployment contains a single Pod that runs the container (it is called __manager__). The deployment ensures that the controller manager Pod is restarted in the event of a failure.

This is what kubebuilder provides as controller scaffolding - it is found in __controllers/fcdinfo_controller.go__. We are most interested in the __FCDInfoReconciler__ function:

```go
func (r *FCDInfoReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
        _ = context.Background()
        _ = r.Log.WithValues("fcdinfo", req.NamespacedName)

        // your logic here

        return ctrl.Result{}, nil
}
```

Considering the business logic that I described above, this is what my updated __FCDInfoReconciler__ function looks like. Hopefully the comments make is easy to understand, but at the end of the day, when this controller gets a reconcile request (something as simple as a get command will trigger this), the status fields of the Custom Resource are updated with the  FCD information from the PV specified in the spec.pvId field. Note that I have omitted a number of required imports that also need to be added to the controller. Refer to the code for the complete [__hostinfo_controller.go__](./controllers/hostinfo_controller.go) code.

First, lets look at the modified FCDInfoReconciler structure, which now has 2 new members representing the different clients, VC1 and VC2.

```go
// FCDInfoReconciler reconciles a FCDInfo object
type FCDInfoReconciler struct {
        client.Client
        VC1    *vim25.Client
        VC2    *govmomi.Client
        Finder *find.Finder
        Log    logr.Logger
        Scheme *runtime.Scheme
}
```

Now lets look at the business logic / Reconcile code. Note that I am switching between the different clients to get different information. In some parts, I am using the vim25 client to get information, and in others I am using the govmomi client to get information. Again, this is just a learning exercise, to show various ways to retrieve vSphere information from an operator. The flow here is that we first get a list of datastores, then we retrieve the FCDs from each of the datastores. If any of the FCDs is a match for the PV we have specificed in our manifest, then we populate the status fields for that PV with the requested information. I have added some additional logging messages to this controller logic, and we can check the manager logs to see these messages later on.

```go
func (r *FCDInfoReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
        ctx := context.Background()
        log := r.Log.WithValues("FCDInfo", req.NamespacedName)

        fcd := &topologyv1.FCDInfo{}
        if err := r.Client.Get(ctx, req.NamespacedName, fcd); err != nil {
                if !k8serr.IsNotFound(err) {
                        log.Error(err, "unable to fetch FCDInfo")
                }
                return ctrl.Result{}, client.IgnoreNotFound(err)
        }

        msg := fmt.Sprintf("received reconcile request for %q (namespace: %q)", fcd.GetName(), fcd.GetNamespace())
        log.Info(msg)

//
// Find the datastores available on this vSphere Infrastructure
//

        dss, err := r.Finder.DatastoreList(ctx, "*")
        if err != nil {
                log.Error(err, "FCDInfo: Could not get datastore list")
                return ctrl.Result{}, err
        } else {
                msg := fmt.Sprintf("FCDInfo: Number of datastores found - %v", len(dss))
                log.Info(msg)

                pc := property.DefaultCollector(r.VC2.Client)
//
// "finder" only lists - to get really detailed info,
// Convert datastores into list of references
//
                var refs []types.ManagedObjectReference
                for _, ds := range dss {
                        refs = append(refs, ds.Reference())
                }

//
// Retrieve name property for all datastore
//

                var dst []mo.Datastore
                err = pc.Retrieve(ctx, refs, []string{"name"}, &dst)
                if err != nil {
                        log.Error(err, "FCDInfo: Could not get datastore info")
                        return ctrl.Result{}, err
                }

                m := vslm.NewObjectManager(r.VC1)

//
// -- Display the FCDs on each datastore (held in array dst)
//

                var objids []types.ID
                var idinfo *types.VStorageObject

                for _, newds := range dst {
                        objids, err = m.List(ctx, newds)
//
// -- With the list of FCD Ids, we can get further information about the FCD retrievec in VStorageObject
//
                        for _, id := range objids {
                                idinfo, err = m.Retrieve(ctx, newds, id.Id)
//
// -- Note the TKGS Guest Clusters have a different PV ID
// -- to the one that is created for them in the Supervisor
// -- This only works for the Supervisor PV ID
//
                                if idinfo.Config.BaseConfigInfo.Name == fcd.Spec.PVId {
                                        msg := fmt.Sprintf("FCDInfo: %v matches %v", idinfo.Config.BaseConfigInfo.Name, fcd.Spec.PVId)
                                        log.Info(msg)

                                        fcd.Status.SizeMB = int64(idinfo.Config.CapacityInMB)

                                        backing := idinfo.Config.BaseConfigInfo.Backing.(*types.BaseConfigInfoDiskFileBackingInfo)
                                        fcd.Status.FilePath = string(backing.FilePath)
                                        fcd.Status.ProvisioningType = string(backing.ProvisioningType)
                                } else {
                                        msg := fmt.Sprintf("FCDInfo: %v does not match %v", idinfo.Config.BaseConfigInfo.Name, fcd.Spec.PVId)
                                        log.Info(msg)
                                }
                        }
                }

                if err := r.Status().Update(ctx, fcd); err != nil {
                        log.Error(err, "unable to update FCDInfo status")
                        return ctrl.Result{}, err
                }
        }

        return ctrl.Result{}, nil
}
```

With the controller logic now in place, we can now proceed to build the controller / manager.

## Step 7 - Build the controller ##

At this point everything is in place to enable us to deploy the controller to the Kubernete cluster. If you remember back to the prerequisites in step 1, we said that you need access to a container image registry, such as __docker.io__ or __quay.io__, or VMware's own [Harbor](https://github.com/goharbor/harbor/blob/master/README.md) registry. This is where we need this access to a registry, as we need to push the controller's container image somewhere that can be accessed from your Kubernetes cluster. In this example, I am using quay.io as my repository.

The __Dockerfile__ with the appropriate directives is already in place to build the container image and include the controller / manager logic. This was once again taken care of by kubebuilder. You must ensure that you login to your image repository, i.e. docker login, before proceeding with the __make__ commands, e.g.

```shell
$ docker login quay.io
Username: cormachogan
Password: ***********
WARNING! Your password will be stored unencrypted in /home/cormac/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
$
```

Next, set an environment variable called __IMG__ to point to your container image repository along with the name and version of the container image, e.g:

```shell
export IMG=quay.io/cormachogan/fcdinfo-controller:v1
```

Next, to create the container image of the controller / manager, and push it to the image container repository in a single step, run the following __make__ command. You could of course run this as two seperate commands as well, ```make docker-build``` followed by ```make docker-push``` if you so wished.

```cmd
make docker-build docker-push IMG=quay.io/cormachogan/fcdinfo-controller:v1
```

The output has been shortened in this example:

```Makefile
$ make docker-build docker-push IMG=quay.io/cormachogan/fcdinfo-controller:v1
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/usr/share/go/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
/usr/share/go/bin/controller-gen "crd:preserveUnknownFields=false,crdVersions=v1,trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
go test ./... -coverprofile cover.out
?       fcdinfo [no test files]
?       fcdinfo/api/v1  [no test files]
ok      fcdinfo/controllers     8.595s  coverage: 0.0% of statements
docker build . -t quay.io/cormachogan/fcdinfo-controller:v1
Sending build context to Docker daemon  40.27MB
Step 1/14 : FROM golang:1.13 as builder
 ---> d6f3656320fe
Step 2/14 : WORKDIR /workspace
 ---> Using cache
 ---> 0f6c055c6fc8
Step 3/14 : COPY go.mod go.mod
 ---> 3312f986aae7
Step 4/14 : COPY go.sum go.sum
 ---> 94b418c8c809
Step 5/14 : RUN go mod download
 ---> Running in 4d8f9baef8dd
go: finding cloud.google.com/go v0.38.0
go: finding github.com/Azure/go-ansiterm v0.0.0-20170929234023-d6e3b3328b78
go: finding github.com/Azure/go-autorest/autorest v0.9.0
go: finding github.com/Azure/go-autorest/autorest/adal v0.5.0
.
. <-- snip!
.
go: finding sigs.k8s.io/controller-runtime v0.5.0
go: finding sigs.k8s.io/structured-merge-diff v1.0.1-0.20191108220359-b1b620dd3f06
go: finding sigs.k8s.io/yaml v1.1.0
Removing intermediate container 4d8f9baef8dd
 ---> 356f55a22080
Step 6/14 : COPY main.go main.go
 ---> 667ef6adcc1b
Step 7/14 : COPY api/ api/
 ---> ec53a81d33ab
Step 8/14 : COPY controllers/ controllers/
 ---> 76d50bddb1f4
Step 9/14 : RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -a -o manager main.go
 ---> Running in 65c6ec99969f
Removing intermediate container 65c6ec99969f
 ---> 06d5809e0c60
Step 10/14 : FROM gcr.io/distroless/static:nonroot
 ---> aa99000bc55d
Step 11/14 : WORKDIR /
 ---> Using cache
 ---> 8bcbc4c15403
Step 12/14 : COPY --from=builder /workspace/manager .
 ---> a3c9917e9534
Step 13/14 : USER nonroot:nonroot
 ---> Running in b7616960444d
Removing intermediate container b7616960444d
 ---> 2f83d33251f3
Step 14/14 : ENTRYPOINT ["/manager"]
 ---> Running in 25e5348f5857
Removing intermediate container 25e5348f5857
 ---> 0d5df043fa77
Successfully built 0d5df043fa77
Successfully tagged quay.io/cormachogan/fcdinfo-controller:v1
docker push quay.io/cormachogan/fcdinfo-controller:v1
The push refers to repository [quay.io/cormachogan/fcdinfo-controller]
8e30502bb918: Pushed
7a5b9c0b4b14: Layer already exists
v1: digest: sha256:f9d0c5242d31dfc71c81201b6bcb27f83e28e213a4b03c20bfca6d6f45388257 size: 739
$
```

The container image of the controller is now built and pushed to the container image registry. But we have not yet deployed it. We have to do one or two further modifications before we take that step.

## Step 8 - Modify the Manager manifest to include environment variables ##

Kubebuilder provides a manager manifest scaffold file for deploying the controller. However, since we need to provide vCenter details to our controller, we need to add these to the controller/manager manifest file. This is found in __config/manager/manager.yaml__. This manifest contains the deployment for the controller. In the spec, we need to add an additional __spec.env__ section which has the environment variables defined, as well as the name of our __secret__ (which we will create shortly). Below is a snippet of that code. Here is the code-complete [config/manager/manager.yaml](./config/manager/manager.yaml)).

```yaml
    spec:
      .
      .
        env:
          - name: GOVMOMI_USERNAME
            valueFrom:
              secretKeyRef:
                name: vc-creds
                key: GOVMOMI_USERNAME
          - name: GOVMOMI_PASSWORD
            valueFrom:
              secretKeyRef:
                name: vc-creds
                key: GOVMOMI_PASSWORD
          - name: GOVMOMI_URL
            valueFrom:
              secretKeyRef:
                name: vc-creds
                key: GOVMOMI_URL
      volumes:
        - name: vc-creds
          secret:
            secretName: vc-creds
      terminationGracePeriodSeconds: 10
```

Note that the __secret__, called __vc-creds__ above, contains the vCenter credentials. This secret needs to be deployed in the same namespace that the controller is going to run in, which is __fcdinfo-system__. Thus, the namespace and secret are created using the following commands, with the environment modified to your own vSphere infrastructure obviously:

```shell
$ kubectl create ns fcdinfo-system
namespace/fcdinfo-system created
```

```shell
$ kubectl create secret generic vc-creds \
--from-literal='GOVMOMI_USERNAME=administrator@vsphere.local' \
--from-literal='GOVMOMI_PASSWORD=VMware123!' \
--from-literal='GOVMOMI_URL=192.168.0.100' \
-n fcdinfo-system
secret/vc-creds created
```

We are now ready to deploy the controller to the Kubernetes cluster.

## Step 9 - Deploy the controller ##

To deploy the controller, we run another __make__ command. This will take care of all of the RBAC, cluster roles and role bindings necessary to run the controller, as well as pinging up the correct image, etc.

```Makefile
make deploy IMG=quay.io/cormachogan/fcdinfo-controller:v1
```

The output looks something like this:

```Makefile
$ make deploy IMG=quay.io/cormachogan/fcdinfo-controller:v1
o: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/usr/share/go/bin/controller-gen "crd:preserveUnknownFields=false,crdVersions=v1,trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
cd config/manager && kustomize edit set image controller=quay.io/cormachogan/fcdinfo-controller:v1
kustomize build config/default | kubectl apply -f -
namespace/fcdinfo-system unchanged
customresourcedefinition.apiextensions.k8s.io/fcdinfoes.topology.corinternal.com configured
role.rbac.authorization.k8s.io/fcdinfo-leader-election-role created
clusterrole.rbac.authorization.k8s.io/fcdinfo-manager-role configured
clusterrole.rbac.authorization.k8s.io/fcdinfo-proxy-role created
clusterrole.rbac.authorization.k8s.io/fcdinfo-metrics-reader created
rolebinding.rbac.authorization.k8s.io/fcdinfo-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/fcdinfo-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/fcdinfo-proxy-rolebinding created
service/fcdinfo-controller-manager-metrics-service created
deployment.apps/fcdinfo-controller-manager created
$
```

## Step 10 - Check controller functionality ##

Now that our controller has been deployed, let's see if it is working. There are a few different commands that we can run to verify the operator is working.

### Step 10.1 - Check the deployment and replicaset ###

The deployment should be READY. Remember to specify the namespace correctly when checking it.

```shell
$ kubectl get rs -n fcdinfo-system
NAME                                    DESIRED   CURRENT   READY   AGE
fcdinfo-controller-manager-566c6fffdb   1         1         1       102s

$ kubectl get deploy -n fcdinfo-system
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
fcdinfo-controller-manager    1/1     1            1           2m8s
```

### Step 10.2 - Check the Pods ###

The deployment manages a single controller Pod. There should be 2 containers READY in the controller Pod. One is the __controller / manager__ and the other is the __kube-rbac-proxy__. The [kube-rbac-proxy](https://github.com/brancz/kube-rbac-proxy/blob/master/README.md) is a small HTTP proxy that can perform RBAC authorization against the Kubernetes API. It restricts requests to authorized Pods only.

```shell
$ kubectl get pods -n fcdinfo-system
NAME                                          READY   STATUS    RESTARTS   AGE
fcdinfo-controller-manager-566c6fffdb-fxjr2   2/2     Running   0          2m39s
```

If you experience issues with the one of the pods not coming online, use the following command to display the Pod status and examine the events.

```shell
kubectl describe pod fcdinfo-controller-manager-566c6fffdb-fxjr2 -n fcdinfo-system
```

### Step 10.3 - Check the controller / manager logs ###

If we query the __logs__ on the manager container, we should be able to observe successful startup messages as well as successful reconcile requests from the FCDInfo CR that we already deployed back in step 5. These reconcile requests should update the __Status__ fields with FCD information as per our controller logic. The command to query the manager container logs in the controller Pod is as follows:

```shell
kubectl logs fcdinfo-controller-manager-566c6fffdb-fxjr2 -n fcdinfo-system manager
```

The output should be somewhat similar to this. Note that there is also a successful __Reconcile__ operation reported, which is good. We can also see some log messages which were added to the controller logic.

```shell
$ kubectl logs fcdinfo-controller-manager-566c6fffdb-fxjr2 -n fcdinfo-system manager
2021-01-26T09:33:12.450Z        INFO    controller-runtime.metrics      metrics server is starting to listen    {"addr": "127.0.0.1:8080"}
2021-01-26T09:33:12.450Z        INFO    setup   starting manager
I0126 09:33:12.451024       1 leaderelection.go:242] attempting to acquire leader lease  fcdinfo-system/b610b79e.corinternal.com...
2021-01-26T09:33:12.451Z        INFO    controller-runtime.manager      starting metrics server {"path": "/metrics"}
I0126 09:33:29.856533       1 leaderelection.go:252] successfully acquired lease fcdinfo-system/b610b79e.corinternal.com
2021-01-26T09:33:29.857Z        INFO    controller-runtime.controller   Starting EventSource    {"controller": "fcdinfo", "source": "kind source: /, Kind="}
2021-01-26T09:33:29.857Z        DEBUG   controller-runtime.manager.events       Normal  {"object": {"kind":"ConfigMap","namespace":"fcdinfo-system","name":"b610b79e.corinternal.com","uid":"9bf00d05-f28b-40ff-97fe-49b0d1a72070","apiVersion":"v1","resourceVersion":"32794039"}, "reason": "LeaderElection", "message": "fcdinfo-controller-manager-566c6fffdb-fxjr2_05075012-2765-4de5-bc7d-d71aa7be687e became leader"}
2021-01-26T09:33:29.957Z        INFO    controller-runtime.controller   Starting Controller     {"controller": "fcdinfo"}
2021-01-26T09:33:29.957Z        INFO    controller-runtime.controller   Starting workers        {"controller": "fcdinfo", "worker count": 1}
2021-01-26T09:33:29.958Z        INFO    controllers.FCDInfo     received reconcile request for "fcdinfo-sample" (namespace: "default")  {"FCDInfo": "default/fcdinfo-sample"}
2021-01-26T09:33:29.963Z        INFO    controllers.FCDInfo     FCDInfo: Number of datastores found - 3 {"FCDInfo": "default/fcdinfo-sample"}
2021-01-26T09:33:31.530Z        DEBUG   controller-runtime.controller   Successfully Reconciled {"controller": "fcdinfo", "request": "default/fcdinfo-sample"}
```

### Step 10.4 - Check if CPU statistics are returned in the status ###

Last but not least, let's see if we can see the CPU information in the __status__ fields of the HostInfo object created earlier.

```yaml
$ kubectl get fcd fcdinfo-sample -o yaml
apiVersion: topology.corinternal.com/v1
kind: FCDInfo
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"topology.corinternal.com/v1","kind":"FCDInfo","metadata":{"annotations":{},"name":"fcdinfo-sample","namespace":"default"},"spec":{"pvId":"pvc-e3f6dd59-cbc0-49a7-97c8-d92a26732c43"}}
  creationTimestamp: "2021-01-25T16:18:20Z"
  generation: 2
  managedFields:
  - apiVersion: topology.corinternal.com/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
      f:spec:
        .: {}
        f:pvId: {}
    manager: kubectl
    operation: Update
    time: "2021-01-26T09:45:27Z"
  - apiVersion: topology.corinternal.com/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        .: {}
        f:filePath: {}
        f:provisioningType: {}
        f:sizeMB: {}
    manager: manager
    operation: Update
    time: "2021-01-26T09:45:28Z"
  name: fcdinfo-sample
  namespace: default
  resourceVersion: "32798540"
  selfLink: /apis/topology.corinternal.com/v1/namespaces/default/fcdinfoes/fcdinfo-sample
  uid: 56bd9a46-2b4d-4ace-843a-38a48b21b547
spec:
  pvId: pvc-e3f6dd59-cbc0-49a7-97c8-d92a26732c43
status:
  filePath: '[vsanDatastore] 038f6b5f-8122-d3af-eabe-246e962c240c/b39bcacc6ff143439f9cd6b7454999e4.vmdk'
  provisioningType: thin
  sizeMB: 1024
```

__Success!!!__ Note that the output above is showing us ```filePath```, ```provisioingType```, and ```sizeMB``` as per our business logic implemented in the controller. How cool is that? You can now go ahead and create additional FCDInfo manifests for different PVs in your Kubernetes environment by specifying different __pvIds__ in the manifest spec, and get information about those First Class Disks as well.

## Cleanup ##

To remove the __fcdinfo__ CR, operator and CRD, run the following commands.

### Remove the FCDInfo CR ###

```shell
$ kubectl delete fcdinfo fcdinfo-sample
fcdinfo.topology.corinternal.com "fcdinfo-sample" deleted
```

### Removed the Operator/Controller deployment ###

Deleting the deployment will removed the ReplicaSet and Pods associated with the controller.

```shell
$ $ kubectl get deploy -n fcdinfo-system
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
fcdinfo-controller-manager   1/1     1            1           17m
```

```shell
$ kubectl delete deploy fcdinfo-controller-manager -n fcdinfo-system
deployment.apps "fcdinfo-controller-manager" deleted
```

### Remove the CRD ###

Next, remove the Custom Resource Definition, __fcdinfoes.topology.corinternal.com__.

```shell
$ kubectl get crds
NAME                                                               CREATED AT
antreaagentinfos.clusterinformation.antrea.tanzu.vmware.com        2020-11-18T17:14:03Z
antreacontrollerinfos.clusterinformation.antrea.tanzu.vmware.com   2020-11-18T17:14:03Z
clusternetworkpolicies.security.antrea.tanzu.vmware.com            2020-11-18T17:14:03Z
fcdinfoes.topology.corinternal.com                                 2021-01-25T13:40:41Z
traceflows.ops.antrea.tanzu.vmware.com                             2020-11-18T17:14:03Z
```

```Makefile
$ make uninstall
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/usr/share/go/bin/controller-gen "crd:preserveUnknownFields=false,crdVersions=v1,trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
kustomize build config/crd | kubectl delete -f -
customresourcedefinition.apiextensions.k8s.io "fcdinfoes.topology.corinternal.com" deleted
```

```shell
$ kubectl get crds
NAME                                                               CREATED AT
antreaagentinfos.clusterinformation.antrea.tanzu.vmware.com        2020-11-18T17:14:03Z
antreacontrollerinfos.clusterinformation.antrea.tanzu.vmware.com   2020-11-18T17:14:03Z
clusternetworkpolicies.security.antrea.tanzu.vmware.com            2020-11-18T17:14:03Z
traceflows.ops.antrea.tanzu.vmware.com                             2020-11-18T17:14:03Z
```

The CRD is now removed. At this point, you can also delete the namespace created for the exercise, in this case __fcdinfo-system__. Removing this namespace will also remove the __vc_creds__ secret created earlier.

## What next? ##

One thing you could do it to extend the __FCDInfo__ fields and Operator logic so that it returns even more information about the FCD. There is a lot of information that can be retrieved via the govmomi API calls.

You can now use __kusomtize__ to package the CRD and controller and distribute it to other Kubernetes clusters. Simply point the __kustomize build__ command at the location of the __kustomize.yaml__ file which is in __config/default__.

```shell
kustomize build config/default/ >> /tmp/fcdinfo.yaml
```

This newly created __fcdinfo.yaml__ manifest includes the CRD, RBAC, Service and Deployment for rolling out the operator on other Kubernetes clusters. Nice, eh?

Finally, if this exercise has given you a desire to do more exciting stuff with Kubernetes Operators when Kubernetes is running on vSphere, check out the [vmGroup](https://github.com/embano1/codeconnect-vm-operator/blob/main/README.md) operator that my colleague __Micheal Gasch__ created. It will let you deploy and manage a set of virtual machines on your vSphere infrastructure via a Kubernetes operator. Cool stuff for sure.
