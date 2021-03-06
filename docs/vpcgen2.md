# Secure a Kubernetes Cluster with VPC Gen2

This tutorial explains how to start to `air-gap` and secure an OpenShift or Kubernetes cluster and physically isolate your cluster using a Virtual Private Cloud (VPC). Using an air gapped cluster is one of the first things you will do to secure your container deployments. For more information about `air-gap`, go [here](airgap.md).

The first step to secure your cluster is to create a Virtual Private Cloud (VPC) and add rules to a `Security Group` of an Application Load Balancer (ALB) to allow certain inbound traffic. You can add a gateway like `API Connect` to the VPC and expose the gateway to manage traffic, for instance by using OAuth, rate limiting and API key access control. 

## Setup

For setup and pre-requisities, go [here](setup2.md).

## About

With Red Hat OpenShift Kubernetes (ROKS) or IBM Cloud Kubernetes Service (IKS) on VPC Generation 2 Computeon IBM Cloud, you can create an OpenShift or Kubernetes cluster in Virtual Private Cloud (VPC) infrastructure in a matter of minutes. OpenShift is a more enterprise-ready secure Container Orchestration (CO) platform, but in this tutorial I use a plain managed Kubernetes service because it is more accessible and affordable. If you want however, you can choose to use OpenShift instead.

This tutorial uses a free IBM Cloud account, a free Pay-As-You-Go account, a free IBM Cloud Kubernetes Service (IKS) service with Kubernetes v1.18 with 1 worker node, and a free Virtual Private Cloud (VPC) service. 

This tutorial is based on the official documentation for [creating a cluster in your Virtual Private Cloud (VPC) on generation 2 compute](https://cloud.ibm.com/docs/containers?topic=containers-vpc_ks_tutorial).

Steps:

1. Setup,
1. Create a VPC Generation 2 Compute using CLI
1. Create a Kubernetes Cluster
1. Deploy the Guestbook Application,
1. Update the Security Group
1. Understanding the Load Balancer
1. Ingress Application Load Balancer (ALB)
1. Cleanup
1. [Optional] Using UI to Create a Cluster with VPC Generation 2 Compute

## Setup

Add the VPC infrastructure service plug-in to the IBM Cloud CLI in the client terminal,

```console
ibmcloud plugin install infrastructure-service
```

verify the installed plugins,

```console
$ ibmcloud plugin list

Listing installed plug-ins...

Plugin Name                                 Version   Status   
cloud-functions/wsk/functions/fn            1.0.46    Update Available   
cloud-object-storage                        1.2.1     Update Available   
container-registry                          0.1.494   Update Available   
container-service/kubernetes-service        1.0.171   Update Available   
vpc-infrastructure/infrastructure-service   0.7.6      
```

Set the following environment variables to be used in the remainder of the tutorial. Set the USERNAME to a unique value shorter than 10 characters and set the IBMID to the email you used to create the IBM Cloud account. The other variables don't need to be changed but you can choose to rename them if you prefer.

```console
USERNAME=<bnewell>
IBMID=<b.newell2@remkoh.dev>

MY_REGION=us-south
MY_ZONE=$MY_REGION-1
MY_VPC_NAME=$USERNAME-vpcgen2-vpc1
MY_VPC_SUBNET_NAME=$USERNAME-vpcsubnet1
MY_PUBLIC_GATEWAY=$USERNAME-public-gateway1
MY_CLUSTER_NAME=$USERNAME-iks118-vpc-cluster1
KS_VERSION=1.18
MY_NAMESPACE=my-guestbook
```

Log in to the IBM Cloud account, if you use a Single Sign-On (SSO) provider, use the --sso flag instead of the username flag.

```console
ibmcloud login -u $IBMID [--sso]
```

Target the region where you want to create your VPC environment. The resource group flag is optional, if you happen to know it, you can target it explicitly, but this tutorial does not use it.

```console
ibmcloud target -r $MY_REGION [-g <resource_group>]
```

The VPC must be set up in the same multi-zone metro location where you want to create your cluster. Target generation 2 infrastructure.

```console
ibmcloud is target --gen 2
```

## Create a VPC Generation 2 Compute using CLI

This section uses the IBM Cloud CLI to create a cluster in your VPC on generation 2 compute.

Create a VPC,

```console
ibmcloud is vpc-create $MY_VPC_NAME
ibmcloud is vpcs

ID    Name    Status    Classic access Default network ACL Default security group Resource group 
r006–3883b334–1f4e-4d6a-b3b7–343a029d81bc remkohdev-vpcgen2-vpc1 available false prowling-prevail-universe-equivocal hatracks-fraying-unloader-prevail default
```

Set an environment variable `MY_VPC_ID`,

```console
MY_VPC_ID=$(ibmcloud is vpcs --output json | jq -r '.[] | select( .name=='\"$MY_VPC_NAME\"') | .id ')
echo $MY_VPC_ID
```

Create a subnet

```console
ibmcloud is subnet-create $MY_VPC_SUBNET_NAME $MY_VPC_ID --zone $MY_ZONE --ipv4-address-count 256
```

Set an environment variable `MY_VPC_SUBNET_ID`,

```console
MY_VPC_SUBNET_ID=$(ibmcloud is subnets --output json | jq -r '.[] | select( .name=='\"$MY_VPC_SUBNET_NAME\"') | .id ')
echo $MY_VPC_SUBNET_ID
```

Inspect the VPC's default security group, set an environment variable `MY_DEFAULT_SG_NAME` and `MY_DEFAULT_SG_ID`,

```console
MY_DEFAULT_SG_NAME=$(ibmcloud is vpcs --output json | jq -r '.[] | select( .name=='\"$MY_VPC_NAME\"') | .default_security_group.name ')
echo $MY_DEFAULT_SG_NAME

MY_DEFAULT_SG_ID=$(ibmcloud is security-groups --output json | jq -r '.[] | select( .name=='\"$MY_DEFAULT_SG_NAME\"') | .id ')
echo $MY_DEFAULT_SG_ID

ibmcloud is security-group $MY_DEFAULT_SG_ID --output json

{
    "created_at": "2020-12-18T22:54:04.000Z",
    "crn": "crn:v1:bluemix:public:is:us-south:a/e65910fa61ce9072d64902d03f3d4774::security-group:r006-e201e996-c849-4d0c-ae41-97e4c303d219",
    "href": "https://us-south.iaas.cloud.ibm.com/v1/security_groups/r006-e201e996-c849-4d0c-ae41-97e4c303d219",
    "id": "r006-e201e996-c849-4d0c-ae41-97e4c303d219",
    "name": "hatracks-fraying-unloader-prevail",
    "network_interfaces": [
        {
            "deleted": {
                "more_info": null
            },
            "href": "https://us-south.iaas.cloud.ibm.com/v1/instances/0717_2ba757dd-0d40-48c6-b72e-6fb2ea12584a/network_interfaces/0717-57f73998-6af6-43e1-981f-698f021f4842",
            "id": "0717-57f73998-6af6-43e1-981f-698f021f4842",
            "name": "unpleased-urethane-recycled-subduing",
            "primary_ipv4_address": "10.240.0.4",
            "resource_type": "network_interface"
        },
        {
            "deleted": {
                "more_info": null
            },
            "href": "https://us-south.iaas.cloud.ibm.com/v1/instances/0717_3affef2d-c4b7-49e0-a785-c65ad2f64a35/network_interfaces/0717-0dc82474-6b74-48e7-89e7-536994a45524",
            "id": "0717-0dc82474-6b74-48e7-89e7-536994a45524",
            "name": "sweep-timpani-unable-stricken",
            "primary_ipv4_address": "10.240.0.5",
            "resource_type": "network_interface"
        }
    ],
    "resource_group": {
        "href": "https://resource-controller.cloud.ibm.com/v2/resource_groups/fdd290732f7d47909181a189494e2990",
        "id": "fdd290732f7d47909181a189494e2990",
        "name": "default"
    },
    "rules": [
        {
            "direction": "outbound",
            "id": "r006-c0b88082-748c-47a8-bc75-c9aabb3da4e3",
            "ip_version": "ipv4",
            "protocol": "all",
            "remote": {
                "cidr_block": "0.0.0.0/0"
            }
        },
        {
            "direction": "inbound",
            "id": "r006-4693fcb0-2998-4d73-b62f-9dd80fe1ac56",
            "ip_version": "ipv4",
            "protocol": "all",
            "remote": {
                "href": "https://us-south.iaas.cloud.ibm.com/v1/security_groups/r006-e201e996-c849-4d0c-ae41-97e4c303d219",
                "id": "r006-e201e996-c849-4d0c-ae41-97e4c303d219",
                "name": "hatracks-fraying-unloader-prevail"
            }
        },
        {
            "direction": "inbound",
            "id": "r006-b0fb17d4-b858-4a1f-bb8c-24231f527155",
            "ip_version": "ipv4",
            "port_max": 22,
            "port_min": 22,
            "protocol": "tcp",
            "remote": {
                "cidr_block": "0.0.0.0/0"
            }
        },
        {
            "direction": "inbound",
            "id": "r006-87a1f11b-6d30-4d63-84df-1bda1c5e25f2",
            "ip_version": "ipv4",
            "protocol": "icmp",
            "remote": {
                "cidr_block": "0.0.0.0/0"
            },
            "type": 8
        },
        {
            "direction": "inbound",
            "id": "r006-ade5f825-49e0-47bf-b6cf-b020b5fa23b8",
            "ip_version": "ipv4",
            "port_max": 80,
            "port_min": 80,
            "protocol": "tcp",
            "remote": {
                "cidr_block": "0.0.0.0/0"
            }
        }
    ],
    "targets": null,
    "vpc": {
        "crn": "crn:v1:bluemix:public:is:us-south:a/e65910fa61ce9072d64902d03f3d4774::vpc:r006-3883b334-1f4e-4d6a-b3b7-343a029d81bc",
        "href": "https://us-south.iaas.cloud.ibm.com/v1/vpcs/r006-3883b334-1f4e-4d6a-b3b7-343a029d81bc",
        "id": "r006-3883b334-1f4e-4d6a-b3b7-343a029d81bc",
        "name": "remkohdev-vpcgen2-vpc1"
    }
}
```

Create a public gateway, and update the subnet with the public gateway, this should create a floating IP for your public gateway attached to your subnet,

```console
ibmcloud is public-gateway-create $MY_PUBLIC_GATEWAY $MY_VPC_ID $MY_ZONE

MY_PUBLIC_GATEWAY_ID=$(ibmcloud is public-gateways --output json | jq -r '.[] | select( .name=='\"$MY_PUBLIC_GATEWAY\"') | .id')
echo $MY_PUBLIC_GATEWAY_ID

ibmcloud is subnet-update $MY_VPC_SUBNET_ID --public-gateway-id $MY_PUBLIC_GATEWAY_ID

MY_FLOATING_IP=$(ibmcloud is subnet-public-gateway $MY_VPC_SUBNET_ID --output json | jq -r '.floating_ip.address')
echo $MY_FLOATING_IP
```

Check that the public gateway is now attached to the subnet by retrieving the public gateway id from the subnet configuration,

```console
MY_PUBLIC_GATEWAY_ID2=$(ibmcloud is subnets --output json | jq -r '.[] | select( .name=='\"$MY_VPC_SUBNET_NAME\"') | .public_gateway.id ')
echo $MY_PUBLIC_GATEWAY_ID2
```

To check the new infrastructure in the IBM Cloud dashboard, go to your [subnets for VPC](https://cloud.ibm.com/vpc-ext/network/subnets), 

![VPC Subnets](images/ibmcloud-vpcext-subnets.png)

Click the linked subnet name to view the new subnet details,

![VPC Subnet Details](images/ibmcloud-subnet-details.png)

You should see a public gateway attached with a floating IP.

## Create a Kubernetes Cluster

Create a cluster in your VPC in the same zone as the subnet. By default, your cluster is created with a public and a private service endpoint. You can use the public service endpoint to access the Kubernetes master,


We already defined environment variables for the region, zone and Kubernetes version, but start to check these are valid and available,

```console
ibmcloud ks zone ls --provider vpc-gen2
ibmcloud ks versions
```

Create the cluster in the VPC,

```console
$ ibmcloud ks cluster create vpc-gen2 --name $MY_CLUSTER_NAME --zone $MY_ZONE --version $KS_VERSION --flavor bx2.2x8 --workers 1 --vpc-id $MY_VPC_ID --subnet-id $MY_VPC_SUBNET_ID

Creating cluster...
OK
Cluster created with ID bvglln7d0e5j0u9lfa80
```

Review your IBM Cloud account resources,

Click the linked cluster name of the cluster you just created. If you do not see the cluster listed yet, wait and refresh the page. Check the status of the new cluster,

```
ibmcloud ks clusters

MY_CLUSTER_ID=$(ibmcloud ks clusters --output json --provider vpc-gen2 | jq -r '.[] | select( .name=='\"$MY_CLUSTER_NAME\"') | .id ')
echo $MY_CLUSTER_ID
```

To continue with the next step, the cluster status and the Ingress status must indicate to be available. The cluster might take about 15 minutes to complete. 

```
ibmcloud ks cluster get --cluster $MY_CLUSTER_ID
```

![IBM Cloud Kubernetes Service Overview](images/ibmcloud-iks-overview.png)

Once the cluster is fully provisioned including the Ingress Application Load Balancer (ALB), you should see the `Worker nodes` status to be `100% Normal`, the `Ingress subdomain` should be populated, among other,

![IBM Cloud Kubernetes Service Overview](images/ibmcloud-iks-overview-done.png)

Or via the CLI,

```console
$ ibmcloud ks cluster get --cluster $MY_CLUSTER_ID
Retrieving cluster c031jqqd0rll2ksfg97g...
OK
                                   
Name:                           bnewell2-iks118-vpc-cluster1   
ID:                             c031jqqd0rll2ksfg97g   
State:                          normal   
Status:                         All Workers Normal   
Created:                        2021-01-18 23:29:47 +0000 (16 hours ago)   
Resource Group ID:              68af6383f717459686457a6434c4d19f   
Resource Group Name:            Default   
Pod Subnet:                     172.17.0.0/18   
Service Subnet:                 172.21.0.0/16   
Workers:                        1   
Worker Zones:                   us-south-1   
Ingress Subdomain:              bnewell2-iks118-vpc-clu-47d7983a425e05fef831e694b7945b16-0000.us-south.containers.appdomain.cloud   
Ingress Secret:                 bnewell2-iks118-vpc-clu-47d7983a425e05fef831e694b7945b16-0000   
Ingress Status:                 warning   
Ingress Message:                One or more ALBs are unhealthy. See http://ibm.biz/ingress-alb-check   
Creator:                        -   
Public Service Endpoint URL:    https://c111.us-south.containers.cloud.ibm.com:31097   
Private Service Endpoint URL:   https://c111.private.us-south.containers.cloud.ibm.com:31097   
Pull Secrets:                   enabled in the default namespace   
VPCs:                           r006-3c9ab19c-e8af-4eb3-ab60-b9777c3cce1c   

Master         
Status:     Ready (15 hours ago)   
State:      deployed   
Health:     normal   
Version:    1.18.14_1537   
Location:   Dallas   
URL:        https://c111.us-south.containers.cloud.ibm.com:31097 
```

You can also check the status of the Application Load Balancer (ALB),

```console
$ ibmcloud ks ingress alb ls --cluster $MY_CLUSTER_ID

OK
ALB ID                                Enabled   State      Type      Load Balancer Hostname                 Zone         Build                                  Status   
private-crc031jqqd0rll2ksfg97g-alb1   false     disabled   private   -                                      us-south-1   ingress:/ingress-auth:                 -   
public-crc031jqqd0rll2ksfg97g-alb1    true      enabled    public    f128a85f-us-south.lb.appdomain.cloud   us-south-1   ingress:0.35.0_869_iks/ingress-auth:   -   
```

## Deploy the Guestbook Application

Now the cluster is fully provisioned successfully, you can connect to your cluster and set the `current-context` of the `kubectl`,

```
ibmcloud ks cluster config --cluster $MY_CLUSTER_ID
kubectl config current-context
```

If you have multiple configuration and contexts, you can easily switch between contexts,

```console
kubectl config use-context $MY_CLUSTER_NAME/$MY_CLUSTER_ID
```

Deploy the guestbook application,

```console
kubectl create namespace $MY_NAMESPACE
kubectl create deployment guestbook --image=ibmcom/guestbook:v1 -n $MY_NAMESPACE
kubectl expose deployment guestbook --type="LoadBalancer" --port=3000 --target-port=3000 -n $MY_NAMESPACE
```

List the created service for guestbook,

```console
$ kubectl get svc -n $MY_NAMESPACE

NAME        TYPE           CLUSTER-IP     EXTERNAL-IP                            PORT(S)          AGE
guestbook   LoadBalancer   172.21.48.26   7a6a66a7-us-south.lb.appdomain.cloud   3000:32219/TCP   31m
```

Create environment variables for the public IP address of the `LoadBalancer` service, the `NodePort`, and the port,

```console
SVC_EXTERNAL_IP=$(kubectl get svc -n $MY_NAMESPACE --output json | jq -r '.items[] | .status.loadBalancer.ingress[0].hostname ')
echo $SVC_EXTERNAL_IP

SVC_NODEPORT=$(kubectl get svc -n $MY_NAMESPACE --output json | jq -r '.items[].spec.ports[] | .nodePort')
echo $SVC_NODEPORT

SVC_PORT=$(kubectl get svc -n $MY_NAMESPACE --output json | jq -r '.items[].spec.ports[] | .port')
echo $SVC_PORT
```

**Note** the external IP address is set to the `hostname` of one of two `Load Balancers for VPC`. See the [Load balancers for VPC](https://cloud.ibm.com/vpc-ext/network/loadBalancers) in the IBM Cloud VPC Infrastructure listing.

![Load balancers fort VPC](images/ibmcloud-loadbalancers-for-vpc.png)

Try to send a request to the guestbook application,

```console
curl http://$SVC_EXTERNAL_IP:$SVC_PORT

curl: (52) Empty reply from server
```

Even if you have created a `LoadBalancer` service with an external IP, the service cannot be reached because the VPC does not include a rule to allow incoming or ingress traffic to the service.

## Update the Security Group

To allow any traffic to applications that are deployed on your cluster's worker nodes, you have to modify the VPC's default security group by ID.

Update the security group and add an inbound rule for the NodePort of the service you created when exposing the guestbook deployment. I only want to allow ingress traffic on the NodePort, so I set the minimum and maximum value of allowed inbound ports to the same NodePort value.

```console
$ ibmcloud is security-group-rule-add $MY_DEFAULT_SG_ID inbound tcp --port-min $SVC_NODEPORT --port-max $SVC_NODEPORT

Creating rule for security group r006-b4f498ea-1e71-489e-95f0-0e64cf8d520f under account Remko de Knikker as user b.newell2@remkoh.dev...
                          
ID                     r006-26b196de-5e3a-4a7d-b5fb-9baf9173feb7   
Direction              inbound   
IP version             ipv4   
Protocol               tcp   
Min destination port   32219   
Max destination port   32219   
Remote                 0.0.0.0/0
```

List all the security group rules,

```console
$ ibmcloud is security-group-rules $MY_DEFAULT_SG_ID

Listing rules of security group r006-b4f498ea-1e71-489e-95f0-0e64cf8d520f under account Remko de Knikker as user b.newell2@remkoh.dev...
ID                                          Direction   IP version   Protocol                        Remote   
r006-6c281868-cfbd-46d1-b714-81e085ae2b85   outbound    ipv4         all                             0.0.0.0/0   
r006-b07bbf77-84a0-4e35-90b7-5422f035cf1e   inbound     ipv4         all                             compacted-imprison-clinic-support   
r006-26b196de-5e3a-4a7d-b5fb-9baf9173feb7   inbound     ipv4         tcp Ports:Min=32219,Max=32219   0.0.0.0/0
```

Or add a security group rule to allow inbound TCP traffic on all Kubernetes ports in the range of 30000–32767.

Try again to reach the guestbook application,

```console
curl http://$SVC_EXTERNAL_IP:$SVC_PORT
```

You should now be able to see the HTML response object from the Guestbook application. Open the guestbook URL in a browser to review the web page.

```console
echo http://$SVC_EXTERNAL_IP:$SVC_PORT
```

![Guestbook v1](images/guestbook-v1.png)

If you want to understand better how the load balancing for VPC works, review the optional extra section [Understanding the Load Balancer for VPC](vpcgen2-loadbalancer.md)

You can try removing the inbound rule again to check if the VPC rejects the request again,

```
ibmcloud is security-group-rule-delete $MY_DEFAULT_SG_ID $MY_DEFAULT_SG_RULE_ID
```

## Conclusion

You are awesome! You secured your Kubernetes cluster with a Virtual Private Cloud (VPC) and started to air-gap the cluster, blocking direct access to your cluster. Security is an important integral part of any software application development and "<i>airgapping</i>" your cluster by adding a VPC Generation 2 is a first step in securing your cluster, network and containers.

## Cleanup

To conclude, you can choose to delete your Kubernetes resources for this tutorial,

```console
$ ibmcloud ks cluster rm --cluster $MY_CLUSTER_NAME

Do you want to delete the persistent storage for this cluster? If yes, the data cannot be recovered. If no, you can delete the persistent storage later in your IBM Cloud infrastructure account. [y/N]> y
After you run this command, the cluster cannot be restored. Remove the cluster remkohdev-iks118-vpc-cluster1? [y/N]> y
Removing cluster remkohdev-iks118-vpc-cluster1, persistent storage...
OK
```

Now the Kubernetes cluster is deleted, we need to first remove the public gateway and load balancers from the subnet, then remove the subnet from the VPC, and delete the VPC.

Delete the Gateways, Load Balancers, Network Interfaces, subnet, public gateways, and finally delete the VPC,

```
ibmcloud is security-group-rules $MY_DEFAULT_SG_ID --output json 

MY_DEFAULT_SG_RULE_ID=$(ibmcloud is security-group-rules $MY_DEFAULT_SG_ID --output json | jq -r --arg SVC_NODEPORT $SVC_NODEPORT  '.[] | select( .port_max==($SVC_NODEPORT|tonumber)) | .id ')
echo $MY_DEFAULT_SG_RULE_ID
```

Delete the rule for the security group,

```
$ ibmcloud is security-group-rule-delete $MY_DEFAULT_SG_ID $MY_DEFAULT_SG_RULE_ID
This will delete security group rule r006-4f5f795f-fa90-44e4-b9c4-cb9457a2a421 and cannot be undone. Continue [y/N] ?> y
Deleting rule r006-4f5f795f-fa90-44e4-b9c4-cb9457a2a421 from security group  r006-db96d593-c224-497d-888c-03d84f6d8e98 under account Remko de Knikker as user b.newell2@remkoh.dev...
OK
Rule r006-4f5f795f-fa90-44e4-b9c4-cb9457a2a421 is deleted.
```

Detach the public gateway from the subnet,

```
$ ibmcloud is subnet-public-gateway-detach $MY_VPC_SUBNET_ID
Detaching public gateway from subnet 0717-57ebaf2d-0de6-4630-af01-6cd84031b679 under account Remko de Knikker as user b.newell2@remkoh.dev...
OK
Public gateway is detached.
```

Delete the public gateway,

```
ibmcloud is public-gateways
PUBLIC_GATEWAY_ID=$(ibmcloud is public-gateways --output json | jq '.[0]' | jq -r '.id' ) 
echo $PUBLIC_GATEWAY_ID

$ ibmcloud is public-gateway-delete $PUBLIC_GATEWAY_ID
This will delete public gateway r006-f4603b78-839b-42a5-949c-76403948821a and cannot be undone. Continue [y/N] ?> y
Deleting public gateway r006-f4603b78-839b-42a5-949c-76403948821a under account Remko de Knikker as user b.newell2@remkoh.dev...
OK
Public gateway r006-f4603b78-839b-42a5-949c-76403948821a is deleted.
```

Delete the subnet,

```
$ ibmcloud is subnet-delete $MY_VPC_SUBNET_ID
This will delete Subnet 0717-57ebaf2d-0de6-4630-af01-6cd84031b679 and cannot be undone. Continue [y/N] ?> y
Deleting subnet 0717-57ebaf2d-0de6-4630-af01-6cd84031b679 under account Remko de Knikker as user b.newell2@remkoh.dev...
OK
Subnet 0717-57ebaf2d-0de6-4630-af01-6cd84031b679 is deleted.
```

Delete the Virtual Private Cloud,

```
MY_VPC_ID=$(ibmcloud is vpcs --output json | jq -r '.[] | select( .name=='\"$MY_VPC_NAME\"') | .id ')
echo $MY_VPC_ID

$ ibmcloud is vpc-delete $MY_VPC_IDThis will delete vpc r006-3c9ab19c-e8af-4eb3-ab60-b9777c3cce1c and cannot be undone. Continue [y/N] ?> y
Deleting vpc r006-3c9ab19c-e8af-4eb3-ab60-b9777c3cce1c under account Remko de Knikker as user b.newell2@remkoh.dev...
OK
vpc r006-3c9ab19c-e8af-4eb3-ab60-b9777c3cce1c is deleted.
```
