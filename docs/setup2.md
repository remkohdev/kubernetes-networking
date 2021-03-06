# Setup

## Pre-requisities

* Free IBM Cloud account, to create a new IBM Cloud account go [here](https://ibm.github.io/workshop-setup/NEWACCOUNT/).
* Free `Pay-As-You-Go` account. To upgrade a free IBM Cloud account, go [here](https://ibm.github.io/workshop-setup/PAYASYOUGO/).
* CognitiveLabs.ai account, to access a client terminal at CognitiveLabs.ai, go [here](https://ibm.github.io/workshop-setup/COGNITIVECLASS/).
* IBM Cloud Kubernetes Service (IKS) v1.18 with a standard plan, a classic provider and at least 2 worker nodes (with a subnet and public IPs, external LoadBalancer (for details about VLAN, subnets and IPs, see [here](https://cloud.ibm.com/docs/containers?topic=containers-subnets) ); **OR** an Red Hat OpenShift Kubernetes Service v4.5 with a classic provider and at least 2 worker nodes.

## Setup Kubernetes Cluster

* To use an IBM provided Kubernetes cluster with your IBM Cloud account and IBMId, grant access permissions to the cluster, as instructed [here](https://ibm.github.io/workshop-setup/GRANTCLUSTER/).
* To connect to a managed `Red Hat OpenShift Kubernetes Service (ROKS)`, go [here](https://ibm.github.io/workshop-setup/ROKS/).

## Setup Client Terminal

This workshop was tested using the Labs environment at CognitiveLabs. To access a client terminal at CognitiveLabs.ai, follow the instructions [here](https://ibm.github.io/workshop-setup/COGNITIVECLASS/). 

## Login to IBM Cloud

1. Log in to your cluster, e.g. if created in the `us-south` region,

    ```console
    IBMID=<your IBMID email>
    ibmcloud login -u $IBMID
    ```

1. If you are using federated SSO login, use the `-sso` flag instead.
1. Select the account in which the cluster was created.

    ![Login to IBM Cloud](images/shell-login-to-cloud.png)

## Connect to Cluster

1. Optionally, if you don't know your cluster name, list all clusters,

    ```console
    KS_NAME_SUB=<substring>
    ibmcloud ks clusters --output json | jq -r 'map(select(.name | contains('\"$KS_NAME_SUB\"')))'
    ```

2. Set the `KS_CLUSTER_NAME` environment variable to the correct cluster name, and the namespace environment variable `$MY_NS` to the desired namespace,

    ```console
    KS_CLUSTER_NAME=<your cluster name>
    MY_NS=my-apps
    ```

3. For IKS, download the cluster configuration to the client,

    ```console
    ibmcloud ks cluster config --cluster $KS_CLUSTER_NAME
    kubectl config current-context
    ```

4. The config should be set to a `clustername/clusterid` pair,
5. For OpenShift, find the `oc login` command with the login token. To connect to a managed `Red Hat OpenShift Kubernetes Service (ROKS)`, go [here](https://ibm.github.io/workshop-setup/ROKS/). For example,

    ```console
    oc login --token=e6PfXsQKjxiUf7qkb4jJxEd851pa0ZuUSEimVuGt4aQ --server=https://c114-e.us-south.containers.cloud.ibm.com:32115
    ```

## Install Calicoctl

**Currently not required**

Install calicoctl,

```console
curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.17.1/calicoctl
chmod +x calicoctl
echo "export PATH=$(pwd):$PATH" > $HOME/.bash_profile
source $HOME/.bash_profile
ibmcloud ks cluster config --cluster $KS_CLUSTER_NAME --admin --network
```

Test the version,

```console
$ calicoctl version

Client Version:    v3.17.1
Git commit:        8871aca3
Cluster Version:   v3.16.5
Cluster Type:      typha,kdd,k8s,operator,openshift,bgp
```

As kubectl plugin,

```console
curl -o kubectl-calico -L  https://github.com/projectcalico/calicoctl/releases/download/v3.17.1/calicoctl
chmod +x kubectl-calico
kubectl calico -h
```