# Kubernetes Multi-cluster Service Discovery using the AWS Cloud Map MCS Controller

> *This repository provides an overview of the Kubernetes Multi-Cluster Service API, and the AWS open source implementation of the mcs-api specification - the [AWS Cloud Map MCS Controller for K8s](https://github.com/aws/aws-cloud-map-mcs-controller-for-k8s). This repository also provides a detailed work instruction and associated artefacts required for the end-end implementation of the AWS Cloud Map MCS Controller across multiple EKS clusters in support of seamless, multi-cluster workload deployments.* 

## Introduction

Kubernetes, with it's implementation of the cluster construct has simplified the ability to schedule workloads across a collection of VMs or nodes. Declarative configuration, immutability, auto-scaling, and self healing have vastly simplified the paradigm of workload management within the cluster - which has enabled teams to move at increasing velocities. 

As the rate of Kubernetes adoption continues to increase, there has been a corresponding increase in the number of use cases that require workloads to break through the perimeter of the single cluster construct. Requirements concerning workload location/proximity, isolation, and reliability have been the primary catalyst for the emergence of deployment scenarios where a single logical workload will span multiple Kubernetes clusters:

- **Location** based concerns include network latency requirements (e.g. bringing the application as close to users as possible), data gravity requirements (e.g. bringing elements of the application as close to fixed data sources as possible), and jurisdiction based requirements (e.g. data residency limitations imposed via governing bodies);
- **Isolation** based concerns include performance (e.g. reduction in "noisy-neighbor" in mixed workload clusters), environmental (e.g. by staged or sandboxed workload constructs such as "dev", "test", and "prod" environments), security (e.g. separating untrusted code or sensitive data), organisational (e.g. teams fall under different business units or management domains), and cost based (e.g. teams are subject to separate budgetary constraints);
- **Reliability** based concerns include blast radius and infrastructure diversity (e.g. preventing an application based or underlying infrastructure issue in one cluster or provider zone from impacting the entire solution), and scale based (e.g. the workload may outgrow a single cluster)
  
  

![alt text](images/deployment-scenario-v0.01.png "Deployment Scenario")



Multi-cluster application architectures tend to be designed to either be **replicated** in nature - with this pattern each participating cluster runs a full copy of each given application; or alternatively they implement more of a **group-by-service** pattern where the services of a single application or system are split or divided amongst multiple clusters.

When it comes to the configuration of Kubernetes (and the surrounding infrastructure) to support a given multi-cluster application architecture - the space has evolved over time to include a number of approaches. Implementations tend draw upon a combination of components at various levels of the stack, and generally speaking they also vary in terms of the "weight" or complexity of the implementation, number and scope of features offered, as well as the associated management overhead. In simple terms these approaches can be loosely grouped into two main categories:

* **Network-centric** approaches focus on network interconnection tooling to implement connectivity between clusters in order to facilitate cross-cluster application communication. The various network-centric approaches include those that are tightly coupled with the CNI (e.g. Cillium Mesh), as well as more CNI agnostic implementations such as Submariner and Skupper. Service mesh implementations also fall into the network-centric category, and these include Istio’s multi-cluster support, Linkerd service mirroring, Kuma from Kong, and Consul’s mesh gateway. There are also various multi-cluster ingress approaches, as well as virtual-kubelet based approaches including Admiralty, Tensile-kube, and Liqo.
* **Kubernetes-centric** approaches focus on supporting and extending the core Kubernetes primitives in order to support multi-cluster use cases. These approaches fall under the stewardship of the Kubernetes [Multicluster Special Interest Group](https://github.com/kubernetes/community/tree/master/sig-multicluster) whose charter is focused on designing, implementing, and maintaining API’s, tools, and documentation related to multi-cluster administration and application management. Subprojects include:
  * **[kubefed](https://github.com/kubernetes-sigs/kubefed)** (Kubernetes Cluster Federation) which implements a mechanism to coordinate the configuration of multiple Kubernetes clusters from a single set of APIs in a hosting cluster. kubefed is considered to be foundational for more complex multi-cluster use cases such as deploying multi-geo applications, and disaster recovery.
  * **[work-api](https://github.com/kubernetes-sigs/work-api)** (Multi-Cluster Works API) aims to group a set of Kubernetes API resources to be applied to one or multiple clusters together as a concept of “work” or “workload” for the purpose of multi-cluster workload lifecycle mangement.
  * **[mcs-api](https://github.com/kubernetes-sigs/mcs-api)** (Multi-cluster Services APIs) implements a new API to extend the single-cluster bounded Kubernetes service concept to function across multiple clusters.

### About the Multi-cluster Service API

Kubernetes' familiar [Service](https://cloud.google.com/kubernetes-engine/docs/concepts/service) object lets you discover and access services within the boundary of a single Kubernetes cluster. The mcs-api implements a Kubernetes-native extension to the Service API, extending the scope of the service resource concept beyond the cluster boundary - providing a mechanism to stitch your multiple clusters together using standard (and familiar) DNS based service discovery.

> *[KEP-1645: Multi-Cluster Services API](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api#kep-1645-multi-cluster-services-api) provides the formal description of the Multi Cluster Service API. KEP-1645 doesn't define a complete implementation - it serves to define how an implementation should behave.At the time of writing the mcs-api version is: `multicluster.k8s.io/v1alpha1`*

The primary deployment scenarios covered by the mcs-api include:

- **Different services each deployed to separate clusters:** I have 2 clusters, each running different services managed by different teams, where services from one team depend on services from the other team. I want to ensure that a service from one team can discover a service from the other team (via DNS resolving to VIP), regardless of the cluster that they reside in. In addition, I want to make sure that if the dependent service is migrated to another cluster, the dependee is not impacted.
- **Single service deployed to multiple clusters:** I have deployed my stateless service to multiple clusters for redundancy or scale. Now I want to propagate topologically-aware service endpoints (local, regional, global) to all clusters, so that other services in my clusters can access instances of this service in priority order based on availability and locality.

The mcs-api is able to support these use cases through the described properties of a "ClusterSet", which is a group of clusters with a high degree of mutual trust and shared ownership that share services amongst themselves - along with two additional API objects: the `ServiceExport` and the `ServiceImport`.

Services are not visible to other clusters in the ClusterSet by default, they must be explicitly marked for export by the user. Creating a `ServiceExport` object for a given service specifies that the service should be exposed across all clusters in the ClusterSet. The mcs-api implementation (typically a controller) will automatically generate a corresponding `ServiceImport` object (which serves as the in-cluster representation of a multi-cluster service) in each importing cluster - for consumer workloads to be able to locate and consume the exported service.

DNS-based service discovery for `ServiceImport` objects is facilitated by the [Kubernetes DNS-Based Multicluster Service Discovery Specification](https://github.com/kubernetes/enhancements/pull/2577) which extends the standard Kubernetes DNS paradigms by implementing records named by service and namespace for `serviceimport` objects, but as differentiated from regular in-cluster DNS service names by using the special zone `.clusterset.local`. I.e. When a `ServiceExport` is created, this will cause a FQDN for the multi-cluster service to become available from within the ClusterSet. The domain name will be of the format `<service>.<ns>.svc.clusterset.local`.

#### AWS Cloud Map MCS Controller for Kubernetes

The [AWS Cloud Map MCS Controller for Kubernetes](https://github.com/aws/aws-cloud-map-mcs-controller-for-k8s) (CMCK) is an open source project that implements the multi-cluster services API specification. 

The CMCK is a controller that syncs services across clusters and makes them available for multi-cluster service discovery and connectivity. The implementation model is decentralsised, and utilises AWS Cloud Map as a central registry for registration and distribution of multi-cluster service data.

#### AWS Cloud Map

[AWS Cloud Map](https://aws.amazon.com/cloud-map) is a cloud resource discovery service that Cloud Map allows applications to discover web-based services via AWS SDK, API calls, or DNS queries. Cloud Map is a fully managed service which eliminates the need to set up, update, and manage your own service discovery tools and software.



## Tutorial

### Overview

Let's consider a deployment scenario where we provision a Service into a single EKS cluster, then make the service available from within a second EKS cluster using the AWS Cloud Map MCS Controller.

> *This tutorial will take you through the end-end implementation of the solution as outlined herein, including a functional implementation of the AWS Cloud Map MCS Controller across x2 EKS clusters situated in separate VPCs.*

#### Solution Baseline

![alt text](images/solution-baseline-v0.01.png "Solution Baseline")



In reference to the **Solution Baseline** diagram:

- We have x2 EKS clusters (cls1 & cls2), each deployed into separate VPCs within a single AWS region.
  - cls1 VPC CIDR: 10.10.0.0/16, Kubernetes service IPv4 CIDR: 172.20.0.0/16
  - cls2 VPC CIDR: 10.12.0.0/16, Kubernetes service IPv4 CIDR: 172.20.0.0/16
- VPC peering is configured to permit network connectivity between workloads within each cluster.
- The CoreDNS multicluster plugin is deployed to each cluster.
- The AWS Cloud Map MCS Controller for Kubernetes is deployed to each cluster.
- Clusters cls1 & cls2 are both configured as members of the same mcs-api clusterset.
- Clusters cls1 & cls2 are both provisioned with the namespace `demo`.
- Cluster cls1 has a `ClusterIP` Service `nginx-hello` deployed to the `demo` namespace which frontends a x3 replica Nginx deployment `nginx-demo`.
  - Service | nginx-hello: 172.20.150.33:80
  - Endpoints | nginx-hello: 10.10.11.140:80,10.10.16.197:80,10.10.22.87:80

#### Service Provisioning

With the required dependencies in place, the admin user is able to create a `ServiceExport` object in cls1 for the `nginx-hello` Service, such that the MCS-Controller implementation will automatically provision a corresponding `ServiceImport` in cls2 for consumer workloads to be able to locate and consume the exported service.



![alt text](images/service-provisioning-v0.01.png "Service Provisioning")

In reference to the **Service Provisioning** diagram:

1. The administrator submits the request to the cls1 Kube API server for a `ServiceExport` object to be created for ClusterIP Service `nginx-hello` in the `demo` Namespace.
2. The MCS-Controller in cls1, watching for `ServiceExport` object creation provisions a corresponding `nginx-hello` service in the Cloud Map `demo` namespace. The Cloud Map service is provisioned with sufficient detail for the Service object and corresponding Endpoint Slice to be provisioned within additional clusters in the ClusterSet.
3. The MCS-Controller in cls2 responds to the creation of the `nginx-hello` Cloud Map Service by provisioning the `ServiceImport` object and corresponding `EndpointSlice` objects via the Kube API Server.
4. The CoreDNS multicluster plugin, watching for `ServiceImport` and `EndpointSlice` creation provisions corresponding DNS records within the `.clusterset.local` zone.

#### Service Consumption

![alt text](images/service-consumption-v0.01.png "Service Consumption")

In reference to the **Service Consumption** diagram:

1. The `client-hello` pod in Cluster 2 needs to consume the `nginx-hello` service, for which all Endpoints are deployed in Cluster 1. The `client-hello` pod requests the resource http://nginx-hello.demo.svc.clusterset.local:80. DNS based service discovery [1b] responds with the IP address of the local `nginx-hello` `ServiceExport` Service `ClusterSetIP`.
2. Requests to the local `ClusterSetIP` at `nginx-hello.demo.svc.clusterset.local` are proxied to the Endpoints located on Cluster 1.

Note: In accordance with the mcs-api specification, a multi-cluster service will be imported by all clusters in which the service's namespace exists, meaning that each exporting cluster will also import the corresponding multi-cluster service. As such, the `nginx-hello` service will also be accessible via `ServiceExport` Service `ClusterSetIP` on Cluster 1. Identical to Cluster 2, the `ServiceExport` Service is resolvable by name at `nginx-hello.demo.svc.clusterset.local`.

### Implementation

### Solution Baseline 

To prepare your environment to match the Solution Baseline deployment scenario, the following prerequisites should be addressed.

#### Clone the `cloud-map-mcs-controller` git repository

Sample configuration files will be used through the course of the tutorial, which have been made available in the `cloud-map-mcs-controller` repository.

Clone the repository to the host on which you will be bootstrapping the clusters:

```bash
https://gitlab.com/byteQualia/cloud-map-mcs-controller.git
```

> *Note: Certain values located within the provided configuration files have been configured for substitution with OS environment variables. Work instructions below will identify which environment variables should be set before issuing any commands which will depend on variable substitution.*

> *Note: All commands as provided should be run from the root directory of the cloned git repository.*

#### Create EKS Clusters

x2 EKS clusters should be provisioned, each deployed into separate VPCs within a single AWS region.

- VPCs and clusters should be provisioned with non-overlapping CIDRs.
- For compatibility with the remainder of the tutorial, it is recommended that `eksctl` be used to provision the clusters and associated security configuration. *By default, the `eksctl create cluster` command will create a dedicated VPC.*

Sample `eksctl` config file `/config/eksctl-cluster.yaml` has been provided:

- Environment variables AWS_REGION, CLUSTER_NAME, NODEGROUP_NAME, and VPC_CIDR should be configured. Example values have been provided in the below command reference - substitute values to suit your preference.
- Example VPC CIDRs match the values provided in the Baseline Configuration description.

Run the following commands to create clusters using `eksctl`. 

Cluster 1:

```bash
cd config
export AWS_REGION=ap-southeast-2
export CLUSTER_NAME=cls1
export NODEGROUP_NAME=cls1-nodegroup1
export VPC_CIDR=10.10.0.0/16
envsubst < eksctl-cluster.yaml | eksctl create cluster -f -
```

Cluster 2:

```bash
cd config
export AWS_REGION=ap-southeast-2
export CLUSTER_NAME=cls2
export NODEGROUP_NAME=cls2-nodegroup1
export VPC_CIDR=10.12.0.0/16
envsubst < eksctl-cluster.yaml | eksctl create cluster -f -
```

#### Create VPC Peering Connection

VPC peering is required to permit network connectivity between workloads provisioned within each cluster.

- To create the VPC Peering connection, follow the instruction [Create a VPC peering connection with another VPC in your account](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html) for guidance.
- VPC route tables in each VPC require updating, follow the instruction [Update your route tables for a VPC peering connection](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-routing.html) for guidance. For simplicity, it's recommended to configure route destinations as the IPv4 CIDR block of the peer VPC.

- Security Groups require updating to permit cross-cluster network communication. EKS cluster security groups in each cluster should be updated to permit inbound traffic originating from external clusters. For simplicity, it's recommended the Cluster 1 & Cluster 2 [EKS Cluster Security groups](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html) be updated to allow inbound traffic from the IPv4 CIDR block of the peer VPC.

> *The [VPC Reachability Analyzer](https://docs.aws.amazon.com/vpc/latest/reachability/getting-started.html) can be used to test and diagnose end-end connectivity between worker nodes within each cluster.*

#### Enable EKS OIDC Provider

In order to map required Cloud Map AWS IAM permissions to the MCS-Controller Kubernetes service account, we need to enable the OpenID Connect (OIDC) identity provider in our EKS clusters using `eksctl`.

- Environment variables REGION and CLUSTERNAME should be configured.

Run the following commands to enable OIDC providers using `eksctl`. 

Cluster 1:

```bash
export AWS_REGION=ap-southeast-2
export CLUSTER_NAME=cls1
eksctl utils associate-iam-oidc-provider \
    --region $AWS_REGION \
    --cluster $CLUSTER_NAME \
    --approve
```

Cluster 2:

```bash
export AWS_REGION=ap-southeast-2
export CLUSTER_NAME=cls2
eksctl utils associate-iam-oidc-provider \
    --region $AWS_REGION \
    --cluster $CLUSTER_NAME \
    --approve
```

#### Implement CoreDNS multicluster plugin

The CoreDNS multicluster plugin implements the [Kubernetes DNS-Based Multicluster Service Discovery Specification](https://github.com/kubernetes/enhancements/pull/2577) which enables CoreDNS to lifecycle manage DNS records for `serviceimprt` objects. To enable the CoreDNS multicluster plugin within both EKS clusters, perform the following procedure.

##### Update CoreDNS RBAC

Run the following command against both clusters to update the `system:coredns` clusterrole to include access to additional multi-cluster API resources:

```bash
kubectl apply -f \config\coredns-clusterrole.yaml
```

##### Update the CoreDNS configmap

Run the following command against both clusters to update the default CoreDNS configmap to include the multicluster plugin directive, and `clusterset.local` zone:

```bash
kubectl apply -f \config\coredns-configmap.yaml
```

##### Update the CoreDNS deployment

Run the following command against both clusters to update the default CoreDNS deployment to use the container image `ghcr.io/aws/aws-cloud-map-mcs-controller-for-k8s/coredns-multicluster/coredns:v1.8.4` - which includes the multicluster plugin:

```bash
kubectl apply -f \config\coredns-deployment.yaml
```

#### Install the aws-cloud-map-mcs-controller-for-k8s

##### Configure MCS-Controller RBAC

Before the Cloud Map MCS-Controller is installed, we will first pre-provision the controller Service Account, granting IAM access rights `AWSCloudMapFullAccess` to ensure that the MCS Controller can lifecycle manage Cloud Map resources.

- Environment variable CLUSTER_NAME should be configured.

Run the following commands to create the MCS-Controller namespace and service accounts in each cluster.

> *Note: Be sure to change the `kubectl` context to the correct cluster before issuing commands.*

Cluster 1:

```bash
export CLUSTER_NAME=cls1
kubectl create namespace cloud-map-mcs-system
eksctl create iamserviceaccount \
--cluster $CLUSTER_NAME \
--namespace cloud-map-mcs-system \
--name cloud-map-mcs-controller-manager \
--attach-policy-arn arn:aws:iam::aws:policy/AWSCloudMapFullAccess \
--override-existing-serviceaccounts \
--approve
```

Cluster 2:

```bash
export CLUSTER_NAME=cls2
kubectl create namespace cloud-map-mcs-system
eksctl create iamserviceaccount \
--cluster $CLUSTER_NAME \
--namespace cloud-map-mcs-system \
--name cloud-map-mcs-controller-manager \
--attach-policy-arn arn:aws:iam::aws:policy/AWSCloudMapFullAccess \
--override-existing-serviceaccounts \
--approve
```

##### Install the MCS-Controller

- Environment variable AWS_REGION should be configured.

Run the following command against both clusters to install the MCS-Controller latest release:

```bash
export AWS_REGION=ap-southeast-2
kubectl apply -k "github.com/aws/aws-cloud-map-mcs-controller-for-k8s/config/controller_install_latest"
```

#### Create `nginx-hello` Service

Now that the clusters, CoreDNS and the MCS-Controller have been configured, we can create the `demo` namespace in both clusters and implement the `nginx-hello` Service and associated Deployment into Cluster 1.

Run the following commands to prepare the demo environment on both clusters.

> *Note: be sure to change the `kubectl` context to the correct cluster before issuing commands.*

Cluster 1:

```bash
kubectl create namespace demo
kubectl apply -f \config\nginx-deployment.yaml
kubectl apply -f \config\nginx-service.yaml
```

Cluster 2:

```bash
kubectl create namespace demo
```

### Service Provisioning

With the Solution Baseline in place, let's continue by implementing the Service Provisioning scenario. We'll create a `ServiceExport` object in Cluster 1 for the `nginx-hello` Service. This will trigger the Cluster 1 MCS-Controller to complete service provisioning and propagation into Cloud Map, and subsequent import and provisioning by the MCS-Controller in Cluster 2.

#### Create `nginx-hello` ServiceExport

Run the following command against Cluster 1 to to create the `ServiceExport` object for the `nginx-hello` Service:

```bash
kubectl apply -f \config\nginx-serviceexport.yaml
```

#### Verify `nginx-hello` ServiceExport

Let's verify the `ServiceExport` creation has succeeded, and that corresponding objects have been created in Cluster 1, Cloud Map, and Cluster 2. 

##### Cluster 1

Inspecting the MCS-Controller logs in Cluster 1, we see that the controller has detected the `ServiceExport` object, and created the corresponding `demo` Namespace and `nginx-hello` Service in Cloud Map: 

```bash
$ kubectl logs cloud-map-mcs-controller-manager-5b9f959fc9-hmz88 -c manager --namespace cloud-map-mcs-system
{"level":"info","ts":1641898137.1080713,"logger":"controllers.ServiceExport","msg":"updating Cloud Map service","namespace":"demo","name":"nginx-hello"}
{"level":"info","ts":1641898137.1081324,"logger":"cloudmap","msg":"fetching a service","namespace":"demo","name":"nginx-hello"}
{"level":"info","ts":1641898137.1082,"logger":"cloudmap","msg":"registering endpoints","namespaceName":"demo","serviceName":"nginx-hello","endpoints":[{"Id":"tcp-10_10_28_116-80","IP":"10.10.28.116","EndpointPort":{"Name":"","Port":80,"TargetPort":"","Protocol":"TCP"},"ServicePort":{"Name":"","Port":80,"TargetPort":"80","Protocol":"TCP"},"Attributes":{"K8S_CONTROLLER":"aws-cloud-map-mcs-controller-for-k8s 97072a6 (97072a6)"}},{"Id":"tcp-10_10_21_133-80","IP":"10.10.21.133","EndpointPort":{"Name":"","Port":80,"TargetPort":"","Protocol":"TCP"},"ServicePort":{"Name":"","Port":80,"TargetPort":"80","Protocol":"TCP"},"Attributes":{"K8S_CONTROLLER":"aws-cloud-map-mcs-controller-for-k8s 97072a6 (97072a6)"}},{"Id":"tcp-10_10_16_120-80","IP":"10.10.16.120","EndpointPort":{"Name":"","Port":80,"TargetPort":"","Protocol":"TCP"},"ServicePort":{"Name":"","Port":80,"TargetPort":"80","Protocol":"TCP"},"Attributes":{"K8S_CONTROLLER":"aws-cloud-map-mcs-controller-for-k8s 97072a6 (97072a6)"}}]}
```

 Using the AWS CLI we can verify Namespace and Service resources provisioned to Cloud Map by the Cluster 1 MCS-Controller:

```bash
$ aws servicediscovery list-namespaces
{
    "Namespaces": [
        {
            "Id": "ns-nlnawwa2wa3ajoh3",
            "Arn": "arn:aws:servicediscovery:ap-southeast-2:911483634971:namespace/ns-nlnawwa2wa3ajoh3",
            "Name": "demo",
            "Type": "HTTP",
            "Properties": {
                "DnsProperties": {
                    "SOA": {}
                },
                "HttpProperties": {
                    "HttpName": "demo"
                }
            },
            "CreateDate": "2022-01-11T08:05:21.815000+00:00"
        }
    ]
}

$ aws servicediscovery list-services
{
    "Services": [
        {
            "Id": "srv-xqirlhajwua5vkvo",
            "Arn": "arn:aws:servicediscovery:ap-southeast-2:911483634971:service/srv-xqirlhajwua5vkvo",
            "Name": "nginx-hello",
            "Type": "HTTP",
            "DnsConfig": {},
            "CreateDate": "2022-01-11T08:05:22.061000+00:00"
        }
    ]
}
$ aws servicediscovery discover-instances --namespace-name demo --service-name nginx-hello
{
    "Instances": [
        {
            "InstanceId": "tcp-10_10_21_133-80",
            "NamespaceName": "demo",
            "ServiceName": "nginx-hello",
            "HealthStatus": "UNKNOWN",
            "Attributes": {
                "AWS_INSTANCE_IPV4": "10.10.21.133",
                "AWS_INSTANCE_PORT": "80",
                "ENDPOINT_PORT_NAME": "",
                "ENDPOINT_PROTOCOL": "TCP",
                "K8S_CONTROLLER": "aws-cloud-map-mcs-controller-for-k8s 97072a6 (97072a6)",
                "SERVICE_PORT": "80",
                "SERVICE_PORT_NAME": "",
                "SERVICE_PROTOCOL": "TCP",
                "SERVICE_TARGET_PORT": "80"
            }
        },
        {
            "InstanceId": "tcp-10_10_28_116-80",
            "NamespaceName": "demo",
            "ServiceName": "nginx-hello",
            "HealthStatus": "UNKNOWN",
            "Attributes": {
                "AWS_INSTANCE_IPV4": "10.10.28.116",
                "AWS_INSTANCE_PORT": "80",
                "ENDPOINT_PORT_NAME": "",
                "ENDPOINT_PROTOCOL": "TCP",
                "K8S_CONTROLLER": "aws-cloud-map-mcs-controller-for-k8s 97072a6 (97072a6)",
                "SERVICE_PORT": "80",
                "SERVICE_PORT_NAME": "",
                "SERVICE_PROTOCOL": "TCP",
                "SERVICE_TARGET_PORT": "80"
            }
        },
        {
            "InstanceId": "tcp-10_10_16_120-80",
            "NamespaceName": "demo",
            "ServiceName": "nginx-hello",
            "HealthStatus": "UNKNOWN",
            "Attributes": {
                "AWS_INSTANCE_IPV4": "10.10.16.120",
                "AWS_INSTANCE_PORT": "80",
                "ENDPOINT_PORT_NAME": "",
                "ENDPOINT_PROTOCOL": "TCP",
                "K8S_CONTROLLER": "aws-cloud-map-mcs-controller-for-k8s 97072a6 (97072a6)",
                "SERVICE_PORT": "80",
                "SERVICE_PORT_NAME": "",
                "SERVICE_PROTOCOL": "TCP",
                "SERVICE_TARGET_PORT": "80"
            }
        }
    ]
}
```

##### Cluster 2

Inspecting the MCS-Controller logs in Cluster 2, we see that the controller has detected the `nginx-hello` Cloud Map Service, and created the corresponding Kubernetes `ServiceImport`:

```bash
$ kubectl logs cloud-map-mcs-controller-manager-5b9f959fc9-v72s4 -c manager --namespace cloud-map-mcs-system
{"level":"info","ts":1641898834.16522,"logger":"controllers.Cloudmap","msg":"created ServiceImport","namespace":"demo","name":"nginx-hello"}
{"level":"error","ts":1641898834.1654398,"logger":"controllers.Cloudmap","msg":"error when syncing service","namespace":"demo","name":"nginx-hello","error":"ServiceImport.multicluster.x-k8s.io \"nginx-hello\" not found","stacktrace":"github.com/go-logr/zapr.(*zapLogger).Error\n\t/go/pkg/mod/github.com/go-logr/zapr@v0.2.0/zapr.go:132\ngithub.com/aws/aws-cloud-map-mcs-controller-for-k8s/pkg/common.logger.Error\n\t/workspace/pkg/common/logger.go:39\ngithub.com/aws/aws-cloud-map-mcs-controller-for-k8s/pkg/controllers.(*CloudMapReconciler).reconcileNamespace\n\t/workspace/pkg/controllers/cloudmap_controller.go:98\ngithub.com/aws/aws-cloud-map-mcs-controller-for-k8s/pkg/controllers.(*CloudMapReconciler).Reconcile\n\t/workspace/pkg/controllers/cloudmap_controller.go:63\ngithub.com/aws/aws-cloud-map-mcs-controller-for-k8s/pkg/controllers.(*CloudMapReconciler).Start\n\t/workspace/pkg/controllers/cloudmap_controller.go:41\nsigs.k8s.io/controller-runtime/pkg/manager.(*controllerManager).startRunnable.func1\n\t/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.8.3/pkg/manager/internal.go:681"}
{"level":"info","ts":1641898836.19138,"logger":"controllers.Cloudmap","msg":"created derived Service","namespace":"demo","name":"imported-lia6jf8qe0"}
{"level":"info","ts":1641898836.20201,"logger":"controllers.Cloudmap","msg":"updated ServiceImport","namespace":"demo","name":"nginx-hello","IP":["172.20.179.134"],"ports":[{"protocol":"TCP","port":80}]}
```

Inspecting the Cluster 2 Kubernetes `ServiceImport` object: 

```bash
$ kubectl get serviceimports.multicluster.x-k8s.io nginx-hello -n demo -o yaml
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  annotations:
    multicluster.k8s.aws/derived-service: imported-lia6jf8qe0
  creationTimestamp: "2022-01-11T11:00:34Z"
  generation: 2
  name: nginx-hello
  namespace: demo
  resourceVersion: "5855"
  uid: 62925de9-2bce-44e5-b6c3-7ca68f1536db
spec:
  ips:
  - 172.20.179.134
  ports:
  - port: 80
    protocol: TCP
  type: ClusterSetIP
status: {}
```

And the corresponding Cluster 2 Kubernetes Endpoint Slice:

```bash
$ kubectl get endpointslices.discovery.k8s.io -n demo
NAME                        ADDRESSTYPE   PORTS   ENDPOINTS                                AGE
imported-lia6jf8qe0-fxppx   IPv4          80      10.10.16.120,10.10.21.133,10.10.28.116   52m
```

Important points to note:

- the `ServiceImport` Service is assigned an IP address from the local Kubernetes service IPv4 CIDR: 172.22.0.0/16 (172.20.179.134) so as to permit service discovery and access to the remote service endpoints from within the local cluster.
- the endpoint IP addresses match those of the `nginx-demo` Endpoints in Cluster 1 (from the Cluster 1 VPC CIDR: 10.10.0.0/16).

### Service Consumption

With the Solution Baseline and Service Provisioning in place, workloads in Cluster 2 are now able to consume the `nginx-hello` Service Endpoints located in Cluster 1 via the locally provisioned `ServiceImport` object. To complete the Service Consumption deployment scenario we'll deploy the `client-hello` Pod into Cluster 2, and observe how it's able to perform cross-cluster service consumption of the `nginx-hello` service in Cluster 1.

#### Create `client-hello` Pod

Run the following command against Cluster 2 create the `client-hello` Pod:

```bash
kubectl apply -f \config\client-hello.yaml
```

#### Verify multi-cluster service consumption

Let's exec into the `client-hello` Pod and perform an `nslookup` to CoreDNS for the `ServiceImport` Service `nginx-hello.demo.svc.clusterset.local`:

```bash
$ kubectl exec -it client-hello -n demo /bin/sh
/ # nslookup nginx-hello.demo.svc.clusterset.local
Server:         172.20.0.10
Address:        172.20.0.10:53

Name:   nginx-hello.demo.svc.clusterset.local
Address: 172.20.179.134
```

Note that the Pod resolves the address of the `ServiceImport` object on Cluster 2.

Finally, generate HTTP requests from the `client-hello` Pod to the `nginx-hello` `ServiceImport` Service:

```bash
/ # curl nginx-hello.demo.svc.clusterset.local
Server address: 10.10.28.116:80
Server name: nginx-demo-59c6cb8d7b-t7q88
Date: 11/Jan/2022:12:07:10 +0000
URI: /
Request ID: 0fd11cf4de48e204a73bacb283afed7a
/ # 
/ # curl nginx-hello.demo.svc.clusterset.local
Server address: 10.10.16.120:80
Server name: nginx-demo-59c6cb8d7b-28n8p
Date: 11/Jan/2022:12:07:28 +0000
URI: /
Request ID: 34a0f7ac5f88b42faeb2de205ca3ff33
/ # 
/ # curl nginx-hello.demo.svc.clusterset.local
Server address: 10.10.21.133:80
Server name: nginx-demo-59c6cb8d7b-tv8hf
Date: 11/Jan/2022:12:07:35 +0000
URI: /
Request ID: cda0f25bcb40ade268bed2c9a9f75e91
```

Note that the responding Server Names and Server addresses are those of the `nginx-demo` Pods on Cluster 1 - confirming that the requests to the local `ClusterSetIP` at `nginx-hello.demo.svc.clusterset.local` on Cluster 2 are proxied cross-cluster to the Endpoints located on Cluster 1!











































kubectl apply -f configmap-coredns.yaml



# Secure workload isolation with Amazon EKS Distro and Kata Containers

## Introduction

Containers have introduced a paradigm shift in how we work with applications, and due to the additional efficiencies in deployment, packaging, and development, the rate of adoption has been skyrocketing.

Many people new to containerisation tend to adopt the mental model that containers are simply a better and faster way of of running virtual machines (VMs). In many respects this analogy holds up (albeit from a very simplistic point of view), however from a security perspective, the two technologies provide a very different posture.

Standard Linux containers allow applications to make system calls directly to the host operating system (OS) kernel, in a similar way that non-containerised applications do - whereas in a VM environment, processes in a virtual machine simply do not have visibility of the host OS kernel.

If you're not running untrusted code in your containers, or hosting a multi-tenant platform; and you've implemented good security practices for the services running within each container, you probably don't need to worry.

But for those of us that are faced with the challenge of needing to running untrusted code in our containers, or perhaps are hosting a multi-tenant platform - providing the highest levels of isolation between workloads in a Kubernetes environment can be challenging.

An effective approach to improve workload isolation is to run each Pod within its own dedicated VM. This provides each Pod with a dedicated hypervisor, OS kernel, memory, and virtualized devices which are completely separate from the host OS. In this deployment scenario, when there's a vulnerability in the containerised workload - the hypervisor within the Pod provides a security boundary which protects the host operating system, as well as other workloads running on the host.

![alt text](images/kata-vs-traditional.png "Kata vs. Traditional containers")  
*Image courtesy of https://katacontainers.io*

If you're running on the AWS cloud, Amazon have made this approach very simple. Scheduling Pods using the managed Kubernetes service [EKS](https://aws.amazon.com/eks) with [Fargate](https://aws.amazon.com/fargate/) actually ensures that each Kubernetes Pod is automatically encapsulated inside it's own dedicated VM. This provides the highest level of isolation for each containerised workload.

If you need to provide a similar level of workload isolation as EKS with Fargate when operating outside of the AWS cloud (e.g. on premises, or at the edge in a hybrid deployment scenario), then [Kata Containers](https://katacontainers.io) is a technology worth considering. Kata Containers is an implementation of a lightweight VM that seamlessly integrates with the container ecosystem, and can be used by Kubernetes to schedule Pods inside of VMs.

The following tutorial will take you through a deployment scenario where we bootstrap a Kubernetes cluster using Amazon EKS Distro (EKS-D), and configure Kubernetes to be capable of scheduling Pods inside VMs using Kata Containers.

This is a deployment pattern that can be adopted to provide a very high degree of workload isolation when provisioning clusters outside of the AWS cloud, for example on-premises, edge locations, or on alternate cloud platforms:

- EKS-D provides the same software that has enabled tens of thousands of Kubernetes clusters on Amazon EKS. This includes the latest upstream updates, as well as extended security patching support.

- In-cluster workload isolation is further enhanced by providing the ability to schedule Pods inside a dedicated VM using Kata Containers.

### About Kata Containers

Kata Containers utilizes open source hypervisors as an isolation boundary for each container (or collection of containers in a Pod).

With Kata Containers, a second layer of isolation is created on top of those provided by traditional namespace containers. The hardware virtualization interface is the basis of this additional layer. Kata launches a lightweight virtual machine, and usees the VM guest’s Linux kernel to create a container workload, or workloads in the case of multi-container Pods. In Kubernetes and in the Kata implementation, the sandbox is implemented at the Pod level. In Kata, this sandbox is created using a virtual machine.

Kata currently supports [multiple hypervisors](https://github.com/kata-containers/kata-containers/blob/main/docs/hypervisors.md), including: QEMU/KVM, Cloud Hypervisor/KVM, and Firecracker/KVM.

#### Kata Containers with Kubernetes

Kubernetes Container Runtime Interface (CRI) implementations allow using any OCI-compatible runtime with Kubernetes, such as the Kata Containers runtime. Kata Containers support both the [CRI-O](https://github.com/kubernetes-incubator/cri-o) and [CRI-containerd](https://github.com/containerd/cri) CRI implementations. 

Kata Containers 1.5 introduced the `shimv2` for containerd 1.2.0, reducing the components required to spawn Pods and containers. This is currently the preferred way to run Kata Containers with Kubernetes.

When configuring Kubernetes to integrate with Kata, typically a Kubernetes [`RuntimeClass`](https://kubernetes.io/docs/concepts/containers/runtime-class/) is created. The RuntimeClass provides the ability to select the container runtime configuration to be used for a given workload via the Pod spec submitted to the Kubernetes API.

![alt text](images/kata-shim-v2.png "Kata Shim V2")  
*Image courtesy of https://katacontainers.io*

#### About Amazon EKS Distro

[Amazon EKS Distro](https://distro.eks.amazonaws.com) is a Kubernetes distribution used by Amazon EKS to help create reliable and secure clusters. EKS Distro includes binaries and containers from open source Kubernetes, etcd (cluster configuration database), networking, storage, and plugins, all tested for compatibility. You can deploy EKS Distro wherever your applications need to run.

You can deploy EKS Distro clusters and let AWS take care of testing and tracking Kubernetes updates, dependencies, and patches. The source code, open source tools, and settings are provided for consistent, reproducible builds. EKS Distro provides extended support for Kubernetes, with builds of previous versions updated with the latest security patches. EKS Distro is available as open source on [GitHub](https://github.com/aws/eks-distro).


## Tutorial

### Overview

This tutorial will guide you through the following procedure:

- Installing Kata Containers onto a bare metal host
- Installing and configuring containerd to integrate with Kata Containers
- Bootstrapping an EKS Disto Kubernetes cluster using kubeadm
- Configuring a Kubernetes RuntimeClass to schedule Pods to Kata VMs running the QEMU/KVM hypervisor

> The example EKS-D cluster deployment uses kubeadm to bring up the control-plane, which may not be your preferred method to bootstrap a cluster in an environment outside of a managed cloud provider. A number of AWS partners are also providing installation support for EKS Distro, including: Canonical (MicroK8s), Kubermatic (KubeOne), Kubestack, Nirmata, Rancher, and Weaveworks. For further information, see the [Partners section](https://distro.eks.amazonaws.com/users/install/partners/) at the EKS Distro website.*

### Prerequisites

Kubeadm is a tool built to provide `kubeadm init` and `kubeadm join` as best-practice "fast paths" for creating Kubernetes clusters.

You will need to use a Linux system that kubeadm supports, as described in the [kubeadm documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) to verify that the system has the required amount of memory, CPU, and other resources.

Kata Containers requires nested virtualization or bare metal. Review the [hardware requirements](https://github.com/kata-containers/kata-containers/blob/main/src/runtime/README.md#hardware-requirements) to see if your system is capable of running Kata Containers.

#### Clone the `eksd-kata-containers` git repository

Sample configuration files will be used through the course of the tutorial, which have been made available within the `eksd-kata-containers` repository.

Clone the eksd-kata-containers repository to the host on which you will be bootstrapping the cluster:

``` bash
git clone https://gitlab.com/byteQualia/eksd-kata-containers.git
```

### Bootstrap the Cluster

Next, we bootstrap the cluster using kubeadm.

#### Prepare the host

Make sure SELinux is disabled by setting SELINUX=disabled in the /etc/sysconfig/selinux file. To turn it off immediately, type:

```bash
sudo setenforce 0
```

Make sure that swap is disabled and that no swap areas are reinstated on reboot. For example, type:

```bash
sudo swapoff -a
```

Permanently disable swap by commenting out or deleting any swap areas in /etc/fstab.

Depending on the exact Linux system you installed, you may need to install additional packages. For example, with an RPM-based (Amazon Linux, CentOS, RHEL or Fedora), ensure that the `iproute-tc`, `socat`, and `conntrack-tools` packages are installed.

To optionally enable a firewall, run the following commands, including opening ports required by Kubernetes:

```bash
sudo yum install firewalld -y
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --zone=public --permanent --add-port=6443/tcp --add-port=2379-2380/tcp --add-port=10250-10252/tcp
```

#### Install container runtime, Kata Containers, and supporting services

Next we need to install a container runtime ([containerd](https://containerd.io) in this example), Kata Containers, and the EKS-D versions of Kubernetes software components.

It's recommended that for production environments both the containerd runtime and Kata Containers are installed using official distribution packages. In this example we will utilise [Kata Manager](https://github.com/kata-containers/kata-containers/blob/main/utils/README.md), which will perform a scripted installation of both components:

```bash
repo="github.com/kata-containers/tests"
go get -d "$repo"
PATH=$PATH:$GOPATH/src/${repo}/cmd/kata-manager
kata-manager.sh install-packages
```

Once installed, update the system path to include Kata binaries:

```bash
sudo su
PATH=$PATH:/opt/kata/bin/
echo "PATH=$PATH:/opt/kata/bin/" >> .profile
exit
```

Verify the host is capable of running Kata Containers:

```bash
kata-runtime kata-check
```

Example output generated on a supported system will read as similar to the following:

```bash
sudo kata-runtime kata-check
WARN[0000] Not running network checks as super user      arch=amd64 name= pid=9064 source=runtime
System is capable of running Kata Containers
System can currently create Kata Containers
```


#### Configure container runtime for Kata

`cri` is a native plugin of containerd 1.1 and above, and it's built into containerd and enabled by default. In order to configure containerd to schedule Kata containers, you need to update the containerd configuration file located at  `/etc/containerd/config.toml` with the following configuration which includes three runtime classes:

- `plugins.cri.containerd.runtimes.runc`: the runc, and it is the default runtime
- `plugins.cri.containerd.runtimes.kata`: The function in containerd (reference [the document here](https://github.com/containerd/containerd/tree/master/runtime/v2#binary-naming)) where the dot-connected string `io.containerd.kata.v2` is translated to `containerd-shim-kata-v2` (i.e. the binary name of the Kata implementation of [Containerd Runtime V2 (Shim API)](https://github.com/containerd/containerd/tree/master/runtime/v2)).
- `plugins.cri.containerd.runtimes.katacli`: the `containerd-shim-runc-v1` calls `kata-runtime`, which is the legacy process.

Example `config.toml`:

```bash
plugins.cri.containerd]
 no_pivot = false
plugins.cri.containerd.runtimes]
 [plugins.cri.containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v1"
    [plugins.cri.containerd.runtimes.runc.options]
      NoPivotRoot = false
      NoNewKeyring = false
      ShimCgroup = ""
      IoUid = 0
      IoGid = 0
      BinaryName = "runc"
      Root = ""
      CriuPath = ""
      SystemdCgroup = false
 [plugins.cri.containerd.runtimes.kata]
    runtime_type = "io.containerd.kata.v2"
[plugins.cri.containerd.runtimes.kata.options]
  ConfigPath = "/opt/kata/share/defaults/kata-containers/configuration-qemu.toml"
 [plugins.cri.containerd.runtimes.katacli]
    runtime_type = "io.containerd.runc.v1"
    [plugins.cri.containerd.runtimes.katacli.options]
      NoPivotRoot = false
      NoNewKeyring = false
      ShimCgroup = ""
      IoUid = 0
      IoGid = 0
      BinaryName = "/opt/kata/bin/kata-runtime"
      Root = ""
      CriuPath = ""
      SystemdCgroup = false
```

From the cloned `eksd-kata-containers` repository, copy the file `/config/config.toml` to 
`/etc/containerd/config.toml` and restart containerd:

```bash
sudo systemctl stop containerd
sudo cp eksd-kata-containers/config/config.toml /etc/containerd/
sudo systemctl start containerd
```

containerd is now able to run containers using the Kata Containers runtime. 

#### Test Kata with containerd

In order to test that containerd can successfully run a Kata container, a shell script named `test-kata.sh` has been provided in the `script` directory within the `eksd-kata-containers` repository.

`test-kata.sh` uses the `ctr` CLI util to pull and run a busybox image as a Kata container, and retrieves the kernel version from within the Kata VM. The script returns both the kernel version reported by busybox from within the Kata VM, as well as the host OS kernel version. Per the sample output, the container kernel (hosted in the VM) is different to the host OS kernel:

```bash
chmod +x eksd-kata-containers/script/check-kata.sh
./eksd-kata-containers/script/check-kata.sh

Testing Kata Containers..

docker.io/library/busybox:latest:                                                 resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:ae39a6f5c07297d7ab64dbd4f82c77c874cc6a94cea29fdec309d0992574b4f7:    exists         |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:1ccc0a0ca577e5fb5a0bdf2150a1a9f842f47c8865e861fa0062c5d343eb8cac: exists         |++++++++++++++++++++++++++++++++++++++|
layer-sha256:f531cdc67389c92deac44e019e7a1b6fba90d1aaa58ae3e8192f0e0eed747152:    exists         |++++++++++++++++++++++++++++++++++++++|
config-sha256:388056c9a6838deea3792e8f00705b35b439cf57b3c9c2634fb4e95cfc896de6:   exists         |++++++++++++++++++++++++++++++++++++++|
elapsed: 2.0 s                                                                    total:   0.0 B (0.0 B/s)
unpacking linux/amd64 sha256:ae39a6f5c07297d7ab64dbd4f82c77c874cc6a94cea29fdec309d0992574b4f7...
done

Test successful:
  Host kernel version      : 4.14.225-169.362.amzn2.x86_64
  Container kernel version : 5.4.71
```

The sample containerd configuration file will direct Kata to use the QEMU/KVM hypervisor, per the `ConfigFile` directive on line 19. Configuration files for Cloud Hypervisor/KVM, and Firecracker/KVM are also installed with Kata Containers:

 - Firecracker: `/opt/kata/share/defaults/kata-containers/configuration-fc.toml`
 - Cloud Hypervisor: `/opt/kata/share/defaults/kata-containers/configuration-clh.toml`

To select an alternate hypervisor, update the ConfigFile directive and restart containerd.

### Prepare Kubernetes environment

Pull and retag the pause, coredns, and etcd containers (copy and paste as one line):

```bash
sudo ctr image pull public.ecr.aws/eks-distro/kubernetes/pause:v1.18.9-eks-1-18-1;\
sudo ctr image pull public.ecr.aws/eks-distro/coredns/coredns:v1.7.0-eks-1-18-1; \
sudo ctr image pull public.ecr.aws/eks-distro/etcd-io/etcd:v3.4.14-eks-1-18-1; \
sudo ctr image tag public.ecr.aws/eks-distro/kubernetes/pause:v1.18.9-eks-1-18-1 public.ecr.aws/eks-distro/kubernetes/pause:3.2; \
sudo ctr image tag public.ecr.aws/eks-distro/coredns/coredns:v1.7.0-eks-1-18-1 public.ecr.aws/eks-distro/kubernetes/coredns:1.6.7; \
sudo ctr image tag public.ecr.aws/eks-distro/etcd-io/etcd:v3.4.14-eks-1-18-1 public.ecr.aws/eks-distro/kubernetes/etcd:3.4.3-0
```

Add the RPM repository to Google cloud RPM packages for Kubernetes by creating the following /etc/yum.repos.d/kubernetes.repo file:

```bash
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
```

Install the required Kubernetes packages:

```bash
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

Load the br_netfilter kernel module, and create /etc/modules-load.d/k8s.conf:

```bash
echo br_netfilter | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe br_netfilter
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

Create the /var/lib/kubelet directory, then configure the /var/lib/kubelet/kubeadm-flags.env file:

```bash
sudo su
mkdir -p /var/lib/kubelet
cat /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd —network-plugin=cni —Pod-infra-container-image=public.ecr.aws/eks-distro/kubernetes/pause:3.2"
exit
```

Get compatible binaries for kubeadm, kubelet, and kubectl:

```bash
cd /usr/bin
sudo rm kubelet kubeadm kubectl
sudo wget https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/kubernetes/v1.18.9/bin/linux/amd64/kubelet; \
sudo wget https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/kubernetes/v1.18.9/bin/linux/amd64/kubeadm; \
sudo wget https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/kubernetes/v1.18.9/bin/linux/amd64/kubectl
sudo chmod +x kubeadm kubectl kubelet
```

Enable the kubelet service:

```bash
sudo systemctl enable kubelet
```

#### Configure kube.yaml

A sample `kube.yaml` file has been provided in the `config` directory within the `eksd-kata-containers` repository.

Update the sample `kube.yaml` by providing the values for variables surrounded by {{ and }} within the `localAPIEndpoint` and `nodeRegistration` sections:

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: {{ primary_ip }}
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: {{ primary_hostname }}
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
```

#### Start the Kubernetes control-plane

Run the `kubeadm init` command, identifying the `config` file as follows:

```bash
sudo kubeadm init --config eksd-kata-containers/config/kube.yaml
...
[init] Using Kubernetes version: v1.18.9-eks-1-18-1
[preflight] Running pre-flight checks
...
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!
```

Your Kubernetes cluster should now be up and running. The kubeadm output shows the exact commands to use to add nodes to the cluster. If something goes wrong, correct the problem and run kubeadm reset to prepare you system to run kubeadm init again.

#### Configure the cluster to schedule Kata Containers

Configure the Kubernetes client locally:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Deploy a Pod network to the cluster. See Installing Addons (https://kubernetes.io/docs/concepts/cluster-administration/addons) for information on available Kubernetes Pod networks. For example, we deploy a Weaveworks network:

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=v1.18.9-eks-1-18-1"
...
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net (http://clusterrole.rbac.authorization.k8s.io/weave-net) created
clusterrolebinding.rbac.authorization.k8s.io/weave-net (http://clusterrolebinding.rbac.authorization.k8s.io/weave-net) created
role.rbac.authorization.k8s.io/weave-net (http://role.rbac.authorization.k8s.io/weave-net) created
rolebinding.rbac.authorization.k8s.io/weave-net (http://rolebinding.rbac.authorization.k8s.io/weave-net) created
daemonset.apps/weave-net created
You can also consider Calico or Cilium networks. Calico is popular because it can be used to propagate routes with BGP, which is often used on-prem.
```

If you are testing with a single node, untaint your master node:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

A sample `runtimeclass.yaml` file has been provided in the `config` directory within the `eksd-kata-containers` repository:

``` yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
```

Create the `kata` RuntimeClass:

```bash
kubectl apply -f eksd-kata-containers/config/runtimeclass.yaml
```

### Schedule Kata Containers with Kubernetes

Sample Pod specs have been provided in the `config` directory within the `eksd-kata-containers` repository.

`nginx-kata.yaml` will schedule a pod within a VM using Kata Containers by specifying `kata` as the `runtimeClassName`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-kata
spec:
  runtimeClassName: kata
  containers:
  - name: nginx
    image: nginx
```

`nginx.yaml` will schedule a pod using the default containerd runtime (runc):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

Schedule the Pods using kubectl:

```bash
kubectl apply -f eksd-kata-containers/config/nginx-kata.yaml
kubectl apply -f eksd-kata-containers/config/nginx.yaml
```

You will now have x2 nginx pods running in the cluster, each using different container runtimes. To validate that the nginx-kata Pod has been scheduled inside a VM, exec into each container and retrieve the kernel version:

```bash
kubectl exec -it nginx-kata -- bash -c "uname -r"
5.4.71
kubectl exec -it nginx -- bash -c "uname -r"
4.14.225-169.362.amzn2.x86_64
```

The `nginx-kata` Pod returns the kernel version reported by the kernel running inside the Kata VM, whereas the `nginx` Pod reports the kernel version of the host OS as it's running as a traditional runc container. 

## Conclusion

The industry shift to containers presents unique challenges in securing user workloads within multi-tenant untrusted environments. 

Kata Containers utilizes open source hypervisors as an isolation boundary for each container (or collection of containers in a pod); this approach solves the shared kernel dilemma with existing bare metal container solutions. 

Combining Kata with EKS-D provides secure VM workload isolation on the same software that has enabled tens of thousands of Kubernetes clusters on Amazon EKS. This includes the latest upstream updates, as well as extended security patching support.
