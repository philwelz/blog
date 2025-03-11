---
title: "Enable Node autoprovisioning (NAP) on an existing AKS cluster"
seoTitle: "Enable Node autoprovisioning (NAP) on an existing AKS cluster"
seoDescription: "This article details the process of enabling Node Autoprovisioning on an existing cluster, alongside the steps for disabling old VMSS based node pools."
datePublished: Fri Feb 09 2024 08:21:55 GMT+0000 (Coordinated Universal Time)
cuid: clsedprmk000o08l16zl10qlb
slug: enable-nap-on-an-existing-aks-cluster
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/npxXWgQ33ZQ/upload/64592a8ee9712f48fd245ad6f71c7afe.jpeg
tags: azure, kubernetes, cloud-native, aks, karpenter

---

Finally [announced](https://azure.microsoft.com/en-us/updates/public-preview-node-autoprovision-support-in-aks/) beginning of December 2023, node autoprovisioning (NAP) enables the use of [Karpenter](https://karpenter.sh/) on your AKS cluster. Usually there are functionality drawbacks when features go in public preview. One of those limitations are that NAP can only be enabled on new clusters.

Due to latest improvements from the Engineering Team that is working on Karpenter at Microsoft, NAP can now be enabled on existing clusters. Follow along to see how your cluster can be migrated to NAP.

> Notice: The only supported network configuration to enable NAP on an existing cluster is Azure Overlay with Cilium data plane.

## Requirements

As mentioned already NAP is currently public preview, so you need to have Azure CLI with the [aks-preview](https://github.com/Azure/azure-cli-extensions/tree/main/src/aks-preview) extension 0.5.170 or later installed.

You can verify if you have the extension installed or if you have to correct version with the following commands:

```bash
# get installed version of aks-preview:
az extension list | grep aks-preview -A3 -B 3

# if you dont have the extension installed, you can install it with:
az extension install --name aks-preview

# update the extension if dont at least match 0.5.170:
az extension update --name aks-preview
```

After that, we need to register the NodeAutoProvisioningPreview feature flag:

```bash
# register the NodeAutoProvisioningPreview feature flag 
az feature register \
  --namespace "Microsoft.ContainerService" \
  --name "NodeAutoProvisioningPreview"

# verify the registration status
az feature show \
  --namespace "Microsoft.ContainerService" \
  --name "NodeAutoProvisioningPreview"

# re-register the Microsoft.ContainerService resource provider
az provider register --namespace Microsoft.ContainerService 
```

## Enable Node autoprovision

To enable NAP on an existing Cluster, simply run this command:

```bash
# export this variables as we will will use them more often
CLUSTER_NAME=aks-nap-test-1
CLUSTER_RG=rg-nap-test

# update existing cluster and enable NAP
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $CLUSTER_RG \
  --node-provisioning-mode Auto
```

After a the update command is successfully finished, we can inspect the Kubernetes API-resources to verify that the Karpenter and the Karpenter Azure provider Custom Resource Definitions (CRDs) were added:

```plaintext
kubectl api-resources | grep -e aksnodeclasses -e nodeclaims -e nodepools

NAME                               SHORTNAMES          APIVERSION                             NAMESPACED   KIND
aksnodeclasses                     aksnc,aksncs        karpenter.azure.com/v1alpha2           false        AKSNodeClass
nodeclaims                                             karpenter.sh/v1beta1                   false        NodeClaim
nodepools                                              karpenter.sh/v1beta1                   false        NodePool
```

Now we are ready to go. Almost! Before disabling the VMSS node pool(s) we will do some precaution checks and verify if the default NAP resources are added to our cluster (you can also deploy your own NodePools and AksNodeClasses at this step if you dont want to stick with the standard):

```plaintext
kubectl get nodepools

NAME           NODECLASS
default        default
system-surge   system-surge


kubectl get aksnodeclasses

NAME           AGE
default        43s
system-surge   43s
```

## Disable the VMSS node pool(s)

Lets also record the nodes and pods that are running on the cluster. As you can see below we are running a system mode and an user mode AKS nodepool with 3 nodes each and all pods of the [inflate deployment](https://github.com/Azure/karpenter-provider-azure/blob/main/examples/workloads/inflate.yaml) are scheduled to the user node pool (as the system node pool has the CriticalAddonsOnly=true:NoSchedule taint):

```plaintext
kubectl get nodes

NAME                                STATUS   ROLES   AGE     VERSION
aks-nodepool1-38290569-vmss000000   Ready    agent   13m     v1.27.7
aks-nodepool1-38290569-vmss000001   Ready    agent   13m     v1.27.7
aks-nodepool1-38290569-vmss000002   Ready    agent   13m     v1.27.7
aks-userpool1-17470761-vmss000000   Ready    agent   4m25s   v1.27.7
aks-userpool1-17470761-vmss000001   Ready    agent   4m25s   v1.27.7
aks-userpool1-17470761-vmss000002   Ready    agent   4m25s   v1.27.7

kubectl get pods -owide

NAME                       READY   STATUS    RESTARTS   AGE     IP             NODE                                NOMINATED NODE   READINESS GATES
inflate-74ccd665f4-7xj2w   1/1     Running   0          6m32s   10.244.5.136   aks-userpool1-17470761-vmss000001   <none>           <none>
inflate-74ccd665f4-bgdjh   1/1     Running   0          114s    10.244.4.212   aks-userpool1-17470761-vmss000000   <none>           <none>
inflate-74ccd665f4-k5mt8   1/1     Running   0          6m32s   10.244.3.128   aks-userpool1-17470761-vmss000002   <none>           <none>
```

Now we are ready to finally disable the user mode node pool(s) by simply scaling it to 0. We could also decide to leave this static node pool(s) as NAP and Karpenter supports this scenario but wont autoscale them. The system node pool has to stay untouched in this scenario.

> If the target node pool(s) have cluster-autoscaler enabled we have to disable it before scaling to 0

```bash
# variable for the VMSS node pool name
NODE_POOL=userpool1

# disable cluster-autoscaler before scaling to 0 if enabled
az aks nodepool update \
  --name $NODE_POOL \
  --name $CLUSTER_NAME \
  --resource-group $CLUSTER_RG \
  --disable-cluster-autoscaler

# scale user node pool to 0
az aks nodepool scale \
  --name $NODE_POOL \
  --name $CLUSTER_NAME \
  --resource-group $CLUSTER_RG \
  --no-wait \
  --node-count 0
```

Now we can let NAP do what NAP does best, choose automatically the best suitable node size for our workload. After 1-2 minutes or workload should be up and running:

```plaintext
kubectl get events -A --field-selector source=karpenter -w

NAMESPACE   LAST SEEN   TYPE     REASON                      OBJECT                         MESSAGE
default     52s         Normal   Nominated                   pod/inflate-74ccd665f4-g2f5b   Pod should schedule on: nodeclaim/default-6hx6j
default     52s         Normal   Nominated                   pod/inflate-74ccd665f4-lt5tf   Pod should schedule on: nodeclaim/default-6hx6j
default     52s         Normal   Nominated                   pod/inflate-74ccd665f4-xbdt6   Pod should schedule on: nodeclaim/default-6hx6j
default     0s          Normal   Unconsolidatable            node/aks-default-6hx6j         Can't replace with a cheaper node

kubectl get nodeclaims

NAME            TYPE               ZONE           NODE                READY   AGE
default-6hx6j   Standard_D4ls_v5   westeurope-2   aks-default-6hx6j   True    5m12s

kubectl get pods -o wide

NAME                       READY   STATUS    RESTARTS   AGE     IP             NODE                NOMINATED NODE   READINESS GATES
inflate-74ccd665f4-g2f5b   1/1     Running   0          5m52s   10.244.3.198   aks-default-6hx6j   <none>           <none>
inflate-74ccd665f4-lt5tf   1/1     Running   0          5m53s   10.244.3.11    aks-default-6hx6j   <none>           <none>
inflate-74ccd665f4-xbdt6   1/1     Running   0          5m53s   10.244.3.2     aks-default-6hx6j   <none>           <none>
```

Voilà! We enabled Node autoprovisioning on our existing AKS cluster. The unused node pools(s) can now safely be removed if wanted:

```bash
# delete unused node pool(s) if wanted
az aks nodepool delete \
  --name $NODE_POOL \
  --name $CLUSTER_NAME \
  --resource-group $CLUSTER_RG \
```

## Final words

Check out the official docs for NAP if you want to have a deeper overview about this new feature. Hopefully the documentation will soon remove the limitation statement that Node autoprovisioning can be only enabled on new clusters. Until that lets spread the word that is possible as of today.

At last I suggest to leave a star for [Karpenter Provider for Azure](https://github.com/Azure/karpenter-provider-azure) on GitHub. Please also help to improve the Karpenter on Azure provider by submitting feedback of issues while using NAP or work on issues labeled with good first issue.

You can catch me on X or LinkedIn if you want to chat. Cheers.