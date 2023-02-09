## Introduction
[Hosted control planes](https://docs.openshift.com/container-platform/4.12/architecture/control-plane.html#hosted-control-planes-overview_control-plane) for Red Hat OpenShift with the KubeVirt provider makes it possible to host OpenShift tenant clusters on bare metal machines at scale. It can be installed on an existing bare metal OpenShift cluster (OCP) environment allowing you to quickly provision multiple guest clusters using KubeVirt virtual machines. The current model allows running hosted control planes and KubeVirt virtual machines on the same underlying base OCP cluster. Unlike the standalone OpenShift cluster where some of the Kubernetes services in the control plane are running as systemd services, the control planes that HyperShift deploy is just another workload which can be scheduled on any available nodes placed in their dedicated namespaces. This post will show the detailed steps of installing HyperShift with the KubeVirt provider on an existing bare metal cluster and configuring the necessary components to launch guest clusters in a matter of minutes.
## Benefits
The list below highlights the benefits of using HyperShift KubeVirt provider:
* Enhance resource utilization by packing multiple hosted control planes and hosted clusters in the same underlying bare metal infrastructure.
* Strong isolation by separating hosted control planes and guest clusters.
* Reduce cluster provision time by eliminating baremetal node bootstrapping process.
## Cluster Preparation
P 4.12.0 is running as the underlying base OCP cluster on top of 6 bare metal nodes (3 masters + 3 workers). Required operators and controllers are listed as follows:
* OpenShift Data Foundation (ODF)  using local storage devices
* OpenShift Virtualization
* MetalLB
* Multicluster Engine
* Cluster Manager
* HyperShift

The list of required operators might vary depending on your specific environment. The following sections cover the detailed configuration and some explanation of each operator and controller, the corresponding installation link is inserted in each section.
#### OpenShift Data Foundation
In our case, [OpenShift Data Foundation](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/4.12/html/deploying_openshift_data_foundation_using_bare_metal_infrastructure/deploy-using-local-storage-devices-bm) (ODF) with local storage devices is used as the default software defined storage to persist the guest cluster etcd pods and VM workers.

After completing the installation guide and creating the corresponding storage system, you should be able to examine the operators installed and the storage classes managed by ODF:
```
[root@e24-h21-740xd ~]# oc get csv -n openshift-storage
NAME                              DISPLAY                       VERSION   REPLACES   PHASE
mcg-operator.v4.12.0              NooBaa Operator               4.12.0               Succeeded
ocs-operator.v4.12.0              OpenShift Container Storage   4.12.0               Succeeded
odf-csi-addons-operator.v4.12.0   CSI Addons                    4.12.0               Succeeded
odf-operator.v4.12.0              OpenShift Data Foundation     4.12.0               Succeeded
```
```
[root@e24-h21-740xd ~]# oc get sc
NAME                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
lvset                         kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  23m
ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   17m
ocs-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  21m
ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   17m
openshift-storage.noobaa.io   openshift-storage.noobaa.io/obc         Delete          Immediate
```
Once ODF is setup, annotate a default storage class for HyperShift to persist VM workers and guest cluster etcd pods:
```
oc patch storageclass ocs-storagecluster-ceph-rbd -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
Check if the storage class is labeled as default:
```
[root@e24-h21-740xd ~]# oc get sc | grep ocs-storagecluster-ceph-rbd
ocs-storagecluster-ceph-rbd (default)   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   20m
```
#### OpenShift Virtualization
[OpenShift Virtualization](https://docs.openshift.com/container-platform/4.12/virt/install/installing-virt-web.html) Operator is an add-on to OCP that allows you to run and manage virtual machines alongside containers. HyperShift with the KubeVirt provider allows you to run guest cluster components using KubeVirt virtual machines. Once you installed the operator and created HyperConverged Object, you should be able to see the following output:
```
[root@e24-h21-740xd ~]# oc get csv -n openshift-cnv
NAME                                       DISPLAY                    VERSION   REPLACES                                   PHASE
kubevirt-hyperconverged-operator.v4.12.0   OpenShift Virtualization   4.12.0    kubevirt-hyperconverged-operator.v4.11.1   Succeeded
```
#### MetalLB
[Metallb](https://docs.openshift.com/container-platform/4.12/networking/metallb/metallb-operator-install.html) is recommended as the network load balancer for bare-metal clusters. Once installation is complete, create the MetalLB instance using the following yaml:
```
oc create -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
EOF
```
An address pool needs to be allocated. In our case, dnsmasq is used, you should be able to see the allocated IP range at /etc/dnsmasq.d/ocp4-lab.conf, please choose a free range and apply the following yaml:
```
oc create -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb
  namespace: metallb-system
spec:
  addresses:
  - 192.168.216.32-192.168.216.122
EOF
```
Once IP address pool is created, we can advertise the address pool using L2 protocol:
```
oc create -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
   - metallb
EOF
```
#### Multicluster Engine
[Multicluster engine](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/multicluster_engine/multicluster_engine_overview#installing-from-the-operatorhub-mce) (MCE) is one of the core components of HyperShift. Make sure to install 2.2.0+ in order to launch 4.12 guest clusters. Once the installation completes, you should be able to see the following output:
```
[root@e24-h21-740xd hypershift]# oc get csv -n multicluster-engine
NAME                                   DISPLAY                              VERSION               REPLACES   PHASE
multicluster-engine.v2.2.0             multicluster engine for Kubernetes   2.2.0                            Succeeded
```
Apply the following yaml to create a MulticlusterEngine instance:
```
oc apply -f - <<EOF
apiVersion: multicluster.openshift.io/v1
kind: MultiClusterEngine
metadata:
  name: multiclusterengine-sample
  spec: {}
EOF
```
Since HyperShift with the KubeVirt provider is in dev preview, the functionality is not enabled by default. Apply the following patch to allow the hypershift addon to be installed in a subsequent step:
```
oc patch mce multiclusterengine-sample --type=merge -p '{"spec":{"overrides":{"components":[{"name":"hypershift-preview","enabled": true}]}}}'
```
#### Cluster Manager
The local-cluster ManagedCluster allows the MCE components to treat the cluster it runs on as a host for guest clusters. Note that the creation of this object might fail initially. This failure occurs if the MultiClusterEngine is still being initialized and hasn’t registered the “ManagedCluster” CRD yet. It might take a few minutes of retrying this command before it succeeds.
```
oc apply -f - <<EOF
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  labels:
    local-cluster: "true"
  name: local-cluster
spec:
  hubAcceptsClient: true
  leaseDurationSeconds: 60
EOF
```
#### HyperShift
Apply the following yaml to enable HyperShift operator within the local cluster:
```
oc apply -f - <<EOF
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: hypershift-addon
  namespace: local-cluster
spec:
  installNamespace: open-cluster-management-agent-addon
EOF
```
The hypershift operator pods can be viewed within the “hypershift” namespace. To verify the operator pod in running in this namespace:
```
[root@e24-h21-740xd ~]# oc get pods -n hypershift | grep "operator.*Running"
operator-78c8bc5898-c47rm   1/1     Running   0          3m54s
```
Ingress wildcard routes are required since the guest cluster's base domain will be a subdomain of the infra cluster's `*apps` A record:
```
oc patch ingresscontroller -n openshift-ingress-operator default --type=json -p '[{ "op": "add", "path": "/spec/routeAdmission", "value": {wildcardPolicy: "WildcardsAllowed"}}]'
```
## Demo
The following sections will take you through the steps of:
* Build HyperShift Client
* Configure Environment Variables
* Create HyperShift KubeVirt Hosted Cluster
* Create Ingress Service
* Create Ingress Route
* Examine Hosted Cluster

#### Build HyperShift Client
To build the HyperShift client binary. Use the following command to build the CLI tool within a pod:
```
podman run --rm --privileged -it -v \
$PWD:/output docker.io/library/golang:1.18 /bin/bash -c \
'git clone https://github.com/openshift/hypershift.git && \
cd hypershift/ && \
make hypershift && \
mv bin/hypershift /output/hypershift'
```
The binary will be placed under the $PWD directory, move the binary to /usr/local/bin which should be included in the $PATH environment variable:
```
sudo mv $PWD/hypershift /usr/local/bin
```
#### Configure Environment Variables
The base domain is retrieved from the ingress configuration and the pull secret is needed for pods accessing images from the registry. Make sure to replace the [PULL_SECRET](https://console.redhat.com/openshift/install/pull-secret) variable with a path to your pull secret file:
```
export KUBEVIRT_CLUSTER_NAME=kv-00
export KUBEVIRT_CLUSTER_NAMESPACE="clusters-${KUBEVIRT_CLUSTER_NAME}"
export BASE_DOMAIN=$(oc get ingresscontroller -n openshift-ingress-operator default -o jsonpath='{.status.domain}')
export KUBEVIRT_CLUSTER_BASE_DOMAIN=${KUBEVIRT_CLUSTER_NAME}.${BASE_DOMAIN}
export PULL_SECRET="/path/to/pull-secret"
```
#### Create HyperShift KubeVirt Hosted Cluster
Use the following command to create a node pool of 2 workers and the guest cluster [playload version](https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/) can be specified by `--release-image`. MCE 2.2 allows us to launch both 4.11 and 4.12 guest clusters:
```
hypershift create cluster \
kubevirt \
--name $KUBEVIRT_CLUSTER_NAME \
--base-domain $BASE_DOMAIN \
--node-pool-replicas=2 \
--memory '8Gi' \
--pull-secret $PULL_SECRET \
--release-image=quay.io/openshift-release-dev/ocp-release@sha256:4c5a7e26d707780be6466ddc9591865beb2e3baa5556432d23e8d57966a2dd18
```
It takes some time for the VMs to be provisioned, you can use the following command to wait until the VM workers reach ready state:
```
oc wait --for=condition=Ready --namespace $KUBEVIRT_CLUSTER_NAMESPACE vm --all --timeout=600s
```
Once the VM workers are ready, we can generate the guest cluster kubeconfig file which is required to retrieve guest cluster ingress NodePort:
```
hypershift create kubeconfig --name="$KUBEVIRT_CLUSTER_NAME" > "${KUBEVIRT_CLUSTER_NAME}-kubeconfig"
```
#### Create Ingress Service
Export the ingress NodePort number to $HTTPS_NODEPORT. Please note that it might take some time for the certificate to be ready:
```
export HTTPS_NODEPORT=$(oc --kubeconfig "${KUBEVIRT_CLUSTER_NAME}-kubeconfig" get services -n openshift-ingress router-nodeport-default -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```
Once we have the NodePort number, we can create the ingress service for the guest cluster:
```
oc create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ${KUBEVIRT_CLUSTER_NAME}
  name: apps-ingress
  namespace: ${KUBEVIRT_CLUSTER_NAMESPACE}
spec:
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: https-443
    port: 443
    protocol: TCP
    targetPort: $HTTPS_NODEPORT
  selector:
    kubevirt.io: virt-launcher
  sessionAffinity: None
  type: ClusterIP
EOF
```
#### Create Ingress Route
```
oc create -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ${KUBEVIRT_CLUSTER_NAME}-443
  namespace: ${KUBEVIRT_CLUSTER_NAMESPACE}
spec:
  host: data.apps.$KUBEVIRT_CLUSTER_BASE_DOMAIN
  wildcardPolicy: Subdomain
  tls:
    termination: passthrough
  port:
    targetPort: https-443
  to:
    kind: Service
    name: apps-ingress
    weight: 100
EOF
```
#### Examine The Hosted Cluster
Once all the operators are deployed successfully, the status of the Available column should be True and the Progress column should be changed to Completed. We created two guest clusters with different releases just to show that It is possible to manage multi-version guest clusters in HyperShift with the KubeVirt provider:
```
[root@e24-h21-740xd hypershift]# oc get hc -A
NAMESPACE   NAME    VERSION   KUBECONFIG               PROGRESS    AVAILABLE   PROGRESSING   MESSAGE
clusters    kv-00   4.12.0    kv-00-admin-kubeconfig   Completed   True        False         The hosted control plane is available
clusters    kv-01   4.11.7    kv-01-admin-kubeconfig   Completed   True        False         The hosted control plane is available
```
## Summary
We went through the detailed steps of installing and configuring necessary operators and controllers to set up HyperShift with the KubeVirt provider in an existing bare metal OCP cluster environment. We also demonstrated how to launch a hosted cluster using the HyperShift command line tool, configuring ingress service and routes and checking the status of the hosted clusters.
## Future Work
Future work will cover the topics of:
* Persistent storage for pod running in the guest cluster
* Different storage configurations for guest etcd pods
* Isolate hosted control planes from nodes that running worker VMs
* Hosted cluster density scale testing


