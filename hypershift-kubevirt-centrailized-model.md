## Introduction
Red Hat HyperShift KubeVirt is another provider makes it possible to host OpenShift tenant clusters on baremetal machines at scale. It can be installed on an existing baremetal OpenShift cluster (OCP) environment allowing you to quickly provision multiple guest clusters in KubeVirt virtual machines. Each guest cluster is associated with a hosted control plane in a single namespace. Unlike the standalone OpenShift cluster, some of the Kubernetes services in the control plane are run as systemd services, whereas the hosted control planes in HyperShift are all run as pods which offers the flexibility to be scheduled in any avaibale nodes. This article will show the detailed steps of installing HyperShift KubeVirt on an existing baremetal cluster and configure the necessary components to launch guest clusters in a matter of minutes.
## Benefits
The list below highlights the benefits of using HyperShift KubeVirt provider:
* Enhance resource consolidation by packing multiple hosted control planes and guest clusters in the same underlying baremetal infrastructure.
* Improve isolation by seperating hosted control planes and guest clusters.
* Reduce cluster provision time by eliminating baremetal node bootstrapping process.
## Lab Environment
OCP 4.11.7 is running as the underlying management cluster on top of 6 baremetal nodes (3 masters + 3 workers). In our case, ODF is used as the default software defined storage to persist the etcd pods and VM workers. The version of each operators is listed as follows:

CNV operators:
```
[root@e24-h21-740xd ~]# oc get csv -n openshift-cnv
NAME                                       DISPLAY                    VERSION               REPLACES                                   PHASE
kubevirt-hyperconverged-operator.v4.11.2   OpenShift Virtualization   4.11.2                kubevirt-hyperconverged-operator.v4.11.1   Succeeded
```
Storage related operators
```
[root@e24-h21-740xd ~]# oc get csv -n openshift-storage
NAME                                   DISPLAY                       VERSION               REPLACES                               PHASE
mcg-operator.v4.11.4                   NooBaa Operator               4.11.4                mcg-operator.v4.11.3                   Succeeded
ocs-operator.v4.11.4                   OpenShift Container Storage   4.11.4                ocs-operator.v4.11.3                   Succeeded
odf-csi-addons-operator.v4.11.4        CSI Addons                    4.11.4                odf-csi-addons-operator.v4.11.3        Succeeded
odf-operator.v4.11.4                   OpenShift Data Foundation     4.11.4                odf-operator.v4.11.3                   Succeeded
```

### Setup
### Demo
## Summary
## Future Work
