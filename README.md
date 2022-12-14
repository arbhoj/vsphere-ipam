# vsphere-ipam

The purpose of this project is to document and demonstrate how static IPPOOLs can be assigned for nodes provisioned via the cluster-api vSphere provisioner (capv) using the [metal3 ipam provider](https://github.com/metal3-io/ip-address-manager.git) and the [vsphere ipam adpater](https://github.com/spectrocloud/cluster-api-provider-vsphere-static-ip.git)
> Note: The goal here is to simplify the steps and reduce dependencies on build tools. For a full deployment please refer to the steps in the respective repositories:
https://github.com/metal3-io/ip-address-manager.git
https://github.com/spectrocloud/cluster-api-provider-vsphere-static-ip.git

## Requirements
1. Cluster API (CAPI) management K8s cluster
> For a [DKP](https://docs.d2iq.com/dkp/2.3) cluster this can be created by running the following commands
```
unset KUBECONFIG
dkp create bootstrap
```
2. Usable Pool of IP Addresses

>The following images are used. Pull them, retag and push to local registry for an airgapped deployment and updated the manifests to reflect the same
> - quay.io/metal3-io/ip-address-manager:main
> - arvindbhoj/capv-static-ip:1.0.0 #This can also be built from scratch from https://github.com/spectrocloud/cluster-api-provider-vsphere-static-ip.git 
> - gcr.io/kubebuilder/kube-rbac-proxy:v0.5.0

## Steps
### Step 1:
Deploy metal3 ipam components to the CAPI cluster

```
kubectl create -f metal3ipam/provider-components/infrastructure-components.yaml
```
> This will create CRD's like ippool, ipaddresses and ipclaims along with the `ipam-controller-manager` deployment for the controller. It uses the `quay.io/metal3-io/ip-address-manager:main` image. Download, retag and push the images to a local registry and change the deployment spec to point to a local image registry for airgapped environments

### Step 2:
Deploy the vsphere ipam adapter

```
kubectl create -f spectro-ipam-adapter/install.yaml
```
>This will create the ipam adapter deployment for capv in the capv-system namespace with the required RBAC. It uses `arvindbhoj/capv-static-ip:1.0.0` and `gcr.io/kubebuilder/kube-rbac-proxy:v0.5.0` images. Download, retag and push the images to a local registry and change the deployment spec to point to a local image registry for airgapped environments

### Step 3:
Define the IP Address range for the cluster being provisioned

>Note: The following is using examples to make it easier to explain what sample values would look like. Modify these as required.

```
export CLUSTER_NAME=dkp-demo
export NETWORK_NAME=Public #This is the name of the network to be used in vSphere
export START_IP=15.235.38.172
export END_IP=15.235.38.176
export CIDR=27
export GATEWAY=15.235.38.190
export DNS_SERVER=8.8.8.8,8.8.4.4

kubectl apply -f - <<EOF
apiVersion: ipam.metal3.io/v1alpha1
kind: IPPool
metadata:
  name: ${CLUSTER_NAME}-pool
  labels:
    cluster.x-k8s.io/network-name: ${NETWORK_NAME}
spec:
  clusterName: ${CLUSTER_NAME}
  namePrefix: ${CLUSTER_NAME}-prov
  pools:
    - start: ${START_IP}
      end: ${END_IP}
      prefix: ${CIDR}
      gateway: ${GATEWAY}
  prefix: 27
  gateway: ${GATEWAY}
  dnsServers: [${DNS_SERVER}]
EOF
```
>Change the IP Pool name, network-name label and ip address pool, gateway and dnsServer details as required.

### Step 4:
Generate the manifests for deploying a vSphere cluster via cluster api. This would be something like this for a [DKP](https://docs.d2iq.com/dkp/2.3/create-new-vsphere-cluster) cluster. 
>Note: The following example is deploying kube-vip to manage the control plane vip and binding it to eth0 interface. If control plane VIP is being managed by an external LB/Proxy, open the generated manifest and delete the kube-vip deployment spec from under the files section of kubeadmcontrolplane. 
```
export CLUSTER_NAME=dkp-demo
export NETWORK=Public
export CONTROL_PLANE_ENDPOINT=xxx.xxx.xxx.xxx
export DATACENTER=dc1
export DATASTORE=datastore_name
export VM_FOLDER=folder_path
export VCENTER=vcenter_host
export SSH_PUB_KEY=path_to_ssh_public_key
export RESOURCE_POOL=vcenter_resource_pool_name
export VCENTER_TEMPLATE=capi_compatible_os_template
dkp create cluster vsphere --cluster-name=${CLUSTER_NAME} --network=${NETWORK} --control-plane-endpoint-host=${CONTROL_PLANE_ENDPOINT} --data-center=${DATACENTER} --data-store=${DATASTORE} --folder=${VM_FOLDER} --server=${VCENTER} --ssh-public-key-file=${SSH_PUB_KEY} --resource-pool=${RESOURCE_POOL} --vm-template=${VCENTER_TEMPLATE} --virtual-ip-interface=eth0 --dry-run -o yaml > dkp-cluster.yaml
```

In the cluster deployment manifest, update the VsphereMachineTemplate resource for the set of nodes that are to source the IP from the defined pool as shown below:
1. Add the `cluster.x-k8s.io/ip-pool-name: ${CLUSTER_NAME}` label. This points to the pool that was created in the last step and ties the MachineTemplate to the pool.
2. Disable dhcp4 and dhcp6

e.g.
```
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: VSphereMachineTemplate
metadata:
  name: dkp-cluster-control-plane
  namespace: default
  labels:
    cluster.x-k8s.io/ip-pool-name: ${CLUSTER_NAME}-pool
spec:
  template:
    spec:
      cloneMode: fullClone
      datacenter: dc1
      datastore: ${DATASTORE}
      diskGiB: 80
      folder: ${VM_FOLDER}
      memoryMiB: 16384
      network:
        devices:
        - dhcp4: false
          dhcp6: false
          networkName: ${NETWORK}
      numCPUs: 4
      resourcePool: ${RESOURCE_POOL}
      server: ${VCENTER}
      template: ${capi_compatible_os_template}
```
### Step 5:

Deploy the cluster by deploying the resources defined in the manifest to the CAPI cluster

e.g.
```
kubectl create -f dkp-cluster.yaml
```

This will deploy a vSphere cluster with the IP's from the range specified in the IPPOOL resource instead of randomly picking an IP from DHCP. 
