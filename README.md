

## Tutorial

### Overview

Let's consider a deployment scenario where we provision a Service into a single EKS cluster, then make the service available from within a second EKS cluster using the AWS Cloud Map MCS Controller.

![alt text](images/solution-baseline-v0.01.png "Solution Baseline")

In reference to the Solution Baseline diagram:

- We have x2 EKS clusters (cls1 & cls2), each deployed into separate VPCs within a single AWS region.
  - cls1 VPC CIDR: 10.10.0.0/16, Kubernetes service IPv4 CIDR: 172.20.0.0/16
  - cls2 VPC CIDR: 10.12.0.0/16, Kubernetes service IPv4 CIDR: 172.20.0.0/16
- VPC peering is configured to permit network connectivity between workloads within each cluster.
- The CoreDNS multicluster plugin is deployed to each cluster.
- The AWS Cloud Map MCS Controller for Kubernetes is deployed to each cluster.
- Clusters cls1 & cls2 are both configured as members of the same mcs-api clusterset.
- Clusters cls1 & cls2 are both provisioned with the namespace `demo`.
- Cluster cls1 has a `ClusterIP` Service `nginx-hello` deployed to the `demo` namespace which frontends a x3 replica Nginx deployment `nginx-demo`.
  - Service | nginx-hello: 172.20.58.137:80
  - Endpoints | nginx-hello: 10.10.11.140:80,10.10.16.197:80,10.10.22.87:80

![alt text](images/service-provisioning-v0.01.png "Service Provisioning")

With the required dependencies in place, the admin user is able to create a `serviceexport` object in cls1 for the `nginx-hello` Service, such that the MCS-Controller implementation will automatically provision a corresponding `serviceimport` in cls2 for consumer workloads to be able to locate and consume the exported service.

In reference to the Service Provisioning diagram:

1. The administrator submits the request to the cls1 Kube API server for a `serviceexport` object to be created for ClusterIP Service `nginx-hello` in the `demo` Namespace.
2. The MCS-Controller in cls1, watching for `serviceexport` object creation provisions a corresponding `nginx-hello` service in the Cloud Map `demo` namespace. The Cloud Map service is provisioned with sufficient detail for the Service object and corresponding Endpoint Slice to be provisioned within additional clusters in the ClusterSet.
3. The MCS-Controller in cls2 responds to the creation of the `nginx-hello` Cloud Map Service by provisioning the `serviceimport` object and corresponding Endpoint Slice objects via the Kube API Server.
4. The CoreDNS multicluster plugin, watching for `serviceimport` and `endpointslice` creation provisions corresponding DNS records within the `.clusterset.local` zone.

![alt text](images/service-consumption-v0.01.png "Service Consumption")In reference to the Service Consumption diagram:

1. The `client-hello` pod in cls2 needs to consume the `nginx-hello` service, which has endpoints deployed in cls1. The `client-hello` pod requests the resource http://nginx-hello.demo.svc.clusterset.local:80. DNS based service discovery [1b] responds with the IP address of the local `nginx-hello` ClusterSetIP Service.

   The `client-hello` pod uses DNS based service discovery [1b] to locate the `nginx-hello` ClusterSetIP Service when requesting the resource `http://nginx-hello.demo.svc.clusterset.local:80`. CoreDNS responds with the local IP address assigned to the `nginx-hello` ClusterSetIP Service.

2. Requests to the local ClusterSetIP Service at `nginx-hello.demo.svc.clusterset.local` are proxied to the Endpoints located on cls1.

Note: In accordance with the mcs-api specification, a multi-cluster service will be imported by all clusters in which the service's namespace exists, meaning that each exporting cluster will also import the corresponding multi-cluster service. As such the `nginx-hello` service will also be accessible via ClusterSetIP Service on cls1 at `nginx-hello.demo.svc.clusterset.local`.

### Solution Baseline

To prepare your environment to match the Solution Baseline deployment scenario, the following prerequisites should be addressed.

#### Clone the `cloud-map-mcs-controller` git repository

Sample configuration files will be used through the course of the tutorial, which have been made available in the `cloud-map-mcs-controller` repository.

Clone the repository to the host on which you will be bootstrapping the cluster:

```bash
https://gitlab.com/byteQualia/cloud-map-mcs-controller.git
```

> Note: Certain values located within the provided configuration files have been configured for substitution with OS environment variables. Work instructions below will identify which environment variables should be set before issuing any commands which will depend on variable substitution.

> Note: All commands should be run from the root directory of the cloned git repository.

#### Create EKS Clusters

x2 EKS clusters should be provisioned, each deployed into separate VPCs within a single AWS region.

- VPCs and clusters should be provisioned with non-overlapping CIDRs.
- For compatibility with the remainder of the tutorial, it is recommended that `eksctl` be used to provision the clusters and associated security configuration. *By default, `eksctl create cluster` will create a dedicated VPC.*

Sample config file `eksctl-cluster.yaml` has been provided in the `config` directory within the `cloud-map-mcs-controller` repository.

- Environment variables AWS_REGION, CLUSTER_NAME, NODEGROUP_NAME, and VPC_CIDR should be configured. Example values have been provided in the below command reference - substitute values to suit your preference.
- Example VPC and Kubernetes service IPv4 CIDRs match the values provided in the Baseline Configuration description.

Run the following commands to create clusters using `eksctl`. 

Cluster 1:

```bash
export AWS_REGION=ap-southeast-2
export CLUSTER_NAME=cls1
export NODEGROUP_NAME=cls1-nodegroup1
export VPC_CIDR=10.10.0.0/16
cd config
envsubst < eksctl-cluster.yaml | eksctl create cluster -f -
cd ..
```

Cluster 2:

```bash
export AWS_REGION=ap-southeast-2
export CLUSTER_NAME=cls2
export NODEGROUP_NAME=cls2-nodegroup1
export VPC_CIDR=10.12.0.0/16
cd config
envsubst < eksctl-cluster.yaml | eksctl create cluster -f -
cd ..
```

#### Create VPC Peering Connection

VPC peering is required to permit network connectivity between workloads provisioned within each cluster.

- To create the VPC Peering connection, follow the instruction [Create a VPC peering connection with another VPC in your account](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html) for guidance.
- VPC route tables in each VPC require updating, follow the instruction [Update your route tables for a VPC peering connection](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-routing.html) for guidance. For simplicity, it's recommended to configure destinations as IPv4 CIDR block of the peer VPC.

- Security Groups require updating to permit cross-cluster network communication. EKS cluster security groups in each cluster should be updated to permit inbound traffic originating from external clusters. For simplicity, it's recommended the Cluster 1 & Cluster 2 [EKS Cluster Security groups](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html) be updated to allow inbound traffic from the IPv4 CIDR block of the peer VPC.

> The [VPC Reachability Analyzer](https://docs.aws.amazon.com/vpc/latest/reachability/getting-started.html) can be used to test and diagnose end-end connectivity between worker nodes within each cluster.

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

Run the following command against both clusters to update the `system:coredns` clusterrole to include access to additional API resources:

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

> Note: Be sure to change the `kubectl` context to the correct cluster before issuing commands.

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
kubectl apply -k "github.com/aws/aws-cloud-map-mcs-controller-for-k8s/config/controller_install_release"
kubectl apply -k "github.com/aws/aws-cloud-map-mcs-controller-for-k8s/config/controller_install_latest"
```

#### Create `nginx-hello` Service

Now that the clusters, CoreDNS and the MCS-Controller have been configured, we can create the `demo` namespace in both clusters, and implement the `nginx-hello` Service and associated Deployment into Cluster 1.

Run the following commands to prepare the demo environment on both clusters.

> Note: be sure to change the `kubectl` context to the correct cluster before issuing commands.

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

With the Solution Baseline in place, let's continue by implementing the Service Provisioning scenario. We'll create a `ServiceExport` object in Cluster 1 for the `nginx-hello` Service. This will trigger the Cluster 1 MCS-Controller service provisioning and propagation into Cloud Map, and subsequent import and provisioning by the MCS-Controller in Cluster 2.

#### Create `nginx-hello` ServiceExport

Run the following command against Cluster 1 to to create the `serviceexport` object for the `nginx-hello` Service:

```bash
kubectl apply -f \config\nginx-serviceexport.yaml
```

#### Verify `nginx-hello` ServiceExport

Let's verify the `serviceexport` creation has succeeded, and that corresponding objects have been created in Cluster 1, Cloud Map, and Cluster 2. 

##### Cluster 1

Inspecting the MCS-Controller logs in Cluster 1, we see that the controller has detected the `ServiceExport` object, and created the corresponding `demo` Namespace and `nginx-hello` Service in Cloud Map: 

```json
kubectl logs cloud-map-mcs-controller-manager-5b9f959fc9-hmz88 -c manager --namespace cloud-map-mcs-system
{"level":"info","ts":1641898137.1080713,"logger":"controllers.ServiceExport","msg":"updating Cloud Map service","namespace":"demo","name":"nginx-hello"}
{"level":"info","ts":1641898137.1081324,"logger":"cloudmap","msg":"fetching a service","namespace":"demo","name":"nginx-hello"}
{"level":"info","ts":1641898137.1082,"logger":"cloudmap","msg":"registering endpoints","namespaceName":"demo","serviceName":"nginx-hello","endpoints":[{"Id":"tcp-10_10_28_116-80","IP":"10.10.28.116","EndpointPort":{"Name":"","Port":80,"TargetPort":"","Protocol":"TCP"},"ServicePort":{"Name":"","Port":80,"TargetPort":"80","Protocol":"TCP"},"Attributes":{"K8S_CONTROLLER":"aws-cloud-map-mcs-controller-for-k8s 97072a6 (97072a6)"}},{"Id":"tcp-10_10_21_133-80","IP":"10.10.21.133","EndpointPort":{"Name":"","Port":80,"TargetPort":"","Protocol":"TCP"},"ServicePort":{"Name":"","Port":80,"TargetPort":"80","Protocol":"TCP"},"Attributes":{"K8S_CONTROLLER":"aws-cloud-map-mcs-controller-for-k8s 97072a6 (97072a6)"}},{"Id":"tcp-10_10_16_120-80","IP":"10.10.16.120","EndpointPort":{"Name":"","Port":80,"TargetPort":"","Protocol":"TCP"},"ServicePort":{"Name":"","Port":80,"TargetPort":"80","Protocol":"TCP"},"Attributes":{"K8S_CONTROLLER":"aws-cloud-map-mcs-controller-for-k8s 97072a6 (97072a6)"}}]}
```

 Using the AWS CLI we can verify Namespace and Service resources provisioned to Cloud Map by the MCS-Controller in Cluster 1:

```yaml
aws servicediscovery list-namespaces 
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
aws servicediscovery list-services
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
aws servicediscovery discover-instances --namespace-name demo --service-name nginx-hello
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

Inspecting the MCS-Controller logs in Cluster 2, we see that the controller has detected the `nginx-hello` Cloud Map service, and created the corresponding `ServiceImport`:

```yaml
kubectl logs cloud-map-mcs-controller-manager-5b9f959fc9-v72s4 -c manager --namespace cloud-map-mcs-system
{"level":"info","ts":1641898834.16522,"logger":"controllers.Cloudmap","msg":"created ServiceImport","namespace":"demo","name":"nginx-hello"}
{"level":"error","ts":1641898834.1654398,"logger":"controllers.Cloudmap","msg":"error when syncing service","namespace":"demo","name":"nginx-hello","error":"ServiceImport.multicluster.x-k8s.io \"nginx-hello\" not found","stacktrace":"github.com/go-logr/zapr.(*zapLogger).Error\n\t/go/pkg/mod/github.com/go-logr/zapr@v0.2.0/zapr.go:132\ngithub.com/aws/aws-cloud-map-mcs-controller-for-k8s/pkg/common.logger.Error\n\t/workspace/pkg/common/logger.go:39\ngithub.com/aws/aws-cloud-map-mcs-controller-for-k8s/pkg/controllers.(*CloudMapReconciler).reconcileNamespace\n\t/workspace/pkg/controllers/cloudmap_controller.go:98\ngithub.com/aws/aws-cloud-map-mcs-controller-for-k8s/pkg/controllers.(*CloudMapReconciler).Reconcile\n\t/workspace/pkg/controllers/cloudmap_controller.go:63\ngithub.com/aws/aws-cloud-map-mcs-controller-for-k8s/pkg/controllers.(*CloudMapReconciler).Start\n\t/workspace/pkg/controllers/cloudmap_controller.go:41\nsigs.k8s.io/controller-runtime/pkg/manager.(*controllerManager).startRunnable.func1\n\t/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.8.3/pkg/manager/internal.go:681"}
{"level":"info","ts":1641898836.19138,"logger":"controllers.Cloudmap","msg":"created derived Service","namespace":"demo","name":"imported-lia6jf8qe0"}
{"level":"info","ts":1641898836.20201,"logger":"controllers.Cloudmap","msg":"updated ServiceImport","namespace":"demo","name":"nginx-hello","IP":["172.20.179.134"],"ports":[{"protocol":"TCP","port":80}]}
```

Looking into the Cluster 2 Kubernetes `ServiceImport` object: 

```yaml
kubectl get serviceimports.multicluster.x-k8s.io nginx-hello -n demo -o yaml
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

And the Cluster 2 Kubernetes Endpoint Slice:

```bash
kubectl get endpointslices.discovery.k8s.io -n demo
NAME                        ADDRESSTYPE   PORTS   ENDPOINTS                                AGE
imported-lia6jf8qe0-fxppx   IPv4          80      10.10.16.120,10.10.21.133,10.10.28.116   52m
```

Important points to note:

- the `ServiceImport` Service is assigned an IP address from the local Kubernetes service IPv4 CIDR: 172.22.0.0/16 (172.22.153.156) so as to permit service discovery and access to the remote service endpoints from within the local cluster.
- the endpoint IP addresses match those of the `nginx-demo` Endpoints in Cluster 1 (i.e. from the cls1 VPC CIDR: 10.10.0.0/16).

### Service Consumption

With the Solution Baseline and Service Provisioning in place, workloads in Cluster 2 are now able to consume the `nginx-hello` Service Endpoints located in Cluster 1 via the locally provisioned `ServiceImport` object. To complete the Service Consumption deployment scenario we'll deploy the `client-hello` Pod into Cluster 2, and observe the cross-cluster service consumption of the `nginx-hello` service.

#### Create `client-hello` Pod

Run the following command against Cluster 2 create the `client-hello` Pod:

```bash
kubectl apply -f \config\client-hello.yaml
```

#### Verify multi-cluster service consumption

Next, exec into a shell in the `client-hello` pod and perform an nslookup for the ServiceImport service `nginx-hello.demo.svc.clusterset.local`:

```
kubectl exec -it client-hello -n demo /bin/sh
/ # nslookup nginx-hello.demo.svc.clusterset.local
Server:         172.20.0.10
Address:        172.20.0.10:53

Name:   nginx-hello.demo.svc.clusterset.local
Address: 172.20.179.134
```

Note that the Pod resolves the address of the `ServiceImport` object on Cluster 2.

Finally, generate HTTP requests to the `ServiceImport` Service:

```
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

Note that the responding Server Names and Server addresses are those of the `nginx-demo` Pods on Cluster 1 - confirming that the requests to the local ClusterSetIP Service at `nginx-hello.demo.svc.clusterset.local` on Cluster 2 are proxied cross-cluster to the Endpoints located on Cluster 1!
