## Introduction
Red Hat HyperShift KubeVirt is another provider makes it possible to host OpenShift control planes on baremetal machines at scale. It can be installed on an existing baremetal OpenShift environment allowing you to quickly provision multiple guest clusters in KubeVirt virtual machines. Each guest cluster is associated with a hosted control plane in a single namespace. Unlike the standalone OpenShift cluster, some of the Kubernetes services in the control plane are run as systemd services, whereas the hosted control planes in HyperShift are all run as pods which offers the flexibility to schedule hosted control planes in any avaibale nodes. This article will show the detailed steps of installing HyperShift KubeVirt on an existing baremetal cluster and configure the necessary components to launch guest clusters in a matter of minutes.
## Benefits
The list below highlights the benefits of using HyperShift KubeVirt provider:
* Enhance resource consolidation by packing multiple hosted control planes and guest clusters in the same underlying baremetal infrastructure.
* Improve isolation by seperating hosted control planes and guest clusters.
* Reduce cluster provision time by eliminating baremetal node bootstrapping process.
## Lab Environment

### Setup
### Demo
## Summary
## Future Work
