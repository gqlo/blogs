HyperShift with the KubeVirt provider Cluster Configuration
===========================================================
## Introduction
This document covers some detailed cluster configurations that you may find useful in a production environment.

## Node Placement and Scheduling
HyperShift with KubeVirt offers excellent flexibility which allows the hosted control plane pods running on the same node where worker VMs are scheduled. It is great for resource ultilization, sometimes you might wanna isolate the hosted control planes in certain infra nodes and place the worker VMs somewhere else. To do that, we can add some node selectors to the HyperConverged Object which allows us to schedule them to the nodes that have matching selectors.

#### Why Patching HyperConverged Object is necessary?
Since hosted control planes are running as pods which does not diffierentiate itself from with the virt-launcher pod. By using node-selector from the HyperShift create command alone is not enough, it will schedule the hosted control plane pods to the designated nodes, but it doesn't prevent virt-launcher pods being scheduled on the same nodes.

#### Patch the HyperConvered Object with two nodeSelectors:
* kubernetes.io/infra-virtualization=virt-components
* kubernetes.io/workers=vm-workload
```
oc patch HyperConverged kubevirt-hyperconverged -n openshift-cnv -p '{"spec":{"infra":{"nodePlacement":{"nodeSelector":{"kubernetes.io/infra-virtualization":"virt-components"}}}}}' --type merge
```

```
oc patch HyperConverged kubevirt-hyperconverged -n openshift-cnv --type=json -p '{"spec":{"workloads":{"nodePlacement":{"nodeSelector":{"kubernetes.io/workers":"vm-workload"}}}}}' --type merge
```

#### Label the Corresponding Nodes:

Worker 011 to 013 for virtualization operator components
```
for i in {011..013}; do oc label node  worker"$i"-1029p kubernetes.io/infra-virtualization=virt-components;done
```
Worker 000-010 for VM workloads
```
for i in {000..004}; do oc label node worker"$i"-1029p kubernetes.io/workers=vm-workload;done
```
Then we can pass the node-Selector param to HyperShift create command:

```
 hypershift create cluster kubevirt \
        --name "$cluster" \
        --node-pool-replicas 3 \
        --control-plane-availability-policy 'HighlyAvailable' \
        --node-selector "kubernetes.io/infra-virtualization=virt-components" \
        --etcd-storage-class "local-worker011-013" \
        --memory '8Gi' \
        --cores '8' \
        --release-image "$image" \
        --pull-secret "$pull_secret"
```
This will place hosted control planes pods on nodes 011-013 and the virt-laucher pods (worker VMs) on nodes 000-010.
#### Zone Config
To make the ectd pods span across 3 different nodes when `HighlyAvaiable` param is passed, we need to add the zone label:
```
for i in {013..011}; do oc label node worker011-1029p topology.kubernetes.io/zone=zone-${i:2:1}=
```
Label the coressponding node with its role just to make more visible when we examine them:
```
for i in {013..011}; do oc label node worker"$i"-1029p node-role.kubernetes.io/zone-${i:2:1}=;done
```

## Storage
There are many storage options avaible..

## Network








