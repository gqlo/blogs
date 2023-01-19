## Introduction
Red Hat HyperShift KubeVirt is another provider that makes it possible to host OpenShift tenant clusters on bare metal machines at scale. It can be installed on an existing bare metal OpenShift cluster (OCP) environment allowing you to quickly provision multiple guest clusters in KubeVirt virtual machines. Each guest cluster is associated with a hosted control plane in a single namespace. Unlike the standalone OpenShift cluster, some of the Kubernetes services in the control plane are run as systemd services, whereas the hosted control planes in HyperShift are all run as pods which offers the flexibility to be scheduled in any available nodes. This article will show the detailed steps of installing HyperShift KubeVirt on an existing bare metal cluster and configure the necessary components to launch guest clusters in a matter of minutes.
## Benefits
The list below highlights the benefits of using HyperShift KubeVirt provider:
* Enhance resource consolidation by packing multiple hosted control planes and guest clusters in the same underlying bare metal infrastructure.
* Improve isolation by separating hosted control planes and guest clusters.
* Reduce cluster provision time by eliminating baremetal node bootstrapping process.
## Lab Environment
OCP 4.11.7 is running as the underlying management cluster on top of 6 bare metal nodes (3 masters + 3 workers). In our case, ODF is used as the default software defined storage to persist the etcd pods and VM workers. Assuming you have both CNV and ODF operators deployed, the version of each operators is listed as follows:

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
Once ODF is properly setup, annotate a default storage class for HyperShift CNV to persist VM workers and etcd pods:
```
oc patch storageclass ocs-storagecluster-ceph-rbd -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
## Other Required Operators
A few other operators are required for bringing up a HyperShift KubeVirt guest cluster, the following section will show you how to install and configure them.
### Multicluster Engine
Multicluster engine is one of the core components of HyperShift. This guide covers the details on how to install MCE via [web console](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/multicluster_engine/multicluster_engine_overview#installing-from-the-operatorhub-mce) or [CLI](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/multicluster_engine/multicluster_engine_overview#installing-from-the-cli-mce).
Once the installation completes, you should be able to see the following status:
```
[root@e24-h21-740xd ~]# oc get csv -n multicluster-engine
NAME                                   DISPLAY                              VERSION               REPLACES                               PHASE
multicluster-engine.v2.1.4             multicluster engine for Kubernetes   2.1.4                 multicluster-engine.v2.1.3             Succeeded
```
Apply the following yaml to create a MulticlusterEngine instance.
```
oc apply -f - <<EOF
apiVersion: multicluster.openshift.io/v1
kind: MultiClusterEngine
metadata:
  name: multiclusterengine-sample
  spec: {}
EOF
```
Since HyperShift 4.11 is in dev preview, the functionality is not enabled by default. Apply the following patch to allow the hypershift addon to be installed in a subsequent step.
```
oc patch mce multiclusterengine-sample --type=merge -p '{"spec":{"overrides":{"components":[{"name":"hypershift-preview","enabled": true}]}}}'
```
### Create Local Managed Cluster
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
### Add HyperShift addons
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
### Verify HyperShift Operator is online.
The hypershift operator pods can be viewed within the “hypershift” namespace. To verify the operator pod in running in this namespace:
```
oc get pods -n hypershift | grep "operator.*Running"
[root@e24-h21-740xd ~]# oc get pods -n hypershift | grep "operator.*Running"
operator-69ccbc88bf-w7sbz   1/1     Running   0          14d
```
### Metallb Operator
Metallb is recommended as the network load balancer for bare-metal clusters. It can be installed via OCP web [console or CLI](https://docs.openshift.com/container-platform/4.11/networking/metallb/metallb-operator-install.html). After the installation completes, proceed the following steps:

#### Create Metallb Instance
```
oc create -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
EOF
```
#### Create Address Pool
In our case, dnsmasq is used, please choose a free range and apply the following yaml:
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
#### Advertise the address pool
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
#### Enable Ingress wildcard routes
Ingress wildcard routes is required since the guest cluster's base domain will be a subdomain of the infra cluster's *apps A record.
```
oc patch ingresscontroller -n openshift-ingress-operator default --type=json -p '[{ "op": "add", "path": "/spec/routeAdmission", "value": {wildcardPolicy: "WildcardsAllowed"}}]'
```
## Build HyperShift Client
One last step is to build the HyperShift Client binary. Use the following command to build the
cli tool within a pod:
```
podman run --rm --privileged -it -v \
$PWD:/output docker.io/library/golang:1.18 /bin/bash -c \
'git clone https://github.com/openshift/hypershift.git && \
cd hypershift/ && \
git checkout release-4.11 && \
make hypershift && \
mv bin/hypershift /output/hypershift'
``` 
The binary will be placed under the $PWD directory, move the binary to /usr/local/bin which should be included in the $PATH environment variable.
```
sudo mv $PWD/hypershift /usr/local/bin
``` 
## Demo
### Environment Variable Setup
Base domain is retrieved from the ingress configuration and the pull secret is needed for pods accessing images from the registry. Make sure to replace the PULL_SECRET variable with a path to your pull secret file.
```
export KUBEVIRT_CLUSTER_NAME=kv-guest-cluster
export KUBEVIRT_CLUSTER_NAMESPACE="clusters-${KUBEVIRT_CLUSTER_NAME}"
export BASE_DOMAIN=$(oc get ingresscontroller -n openshift-ingress-operator default -o jsonpath='{.status.domain}')
export KUBEVIRT_CLUSTER_BASE_DOMAIN=${KUBEVIRT_CLUSTER_NAME}.${BASE_DOMAIN}
export PULL_SECRET="/path/to/pull-secret"
```
### Create HyperShift KubeVirt Guest Cluster
Use the following command to create a node pool of 2 workers and make sure to match the [playload version](https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/) specified by `--release-image`, in our case, OCP 4.11.
```
hypershift create cluster \
kubevirt \
--name $KUBEVIRT_CLUSTER_NAME \
--base-domain $BASE_DOMAIN \
--node-pool-replicas=2 \
--memory '6Gi' \
--pull-secret $PULL_SECRET \
--release-image=quay.io/openshift-release-dev/ocp-release@sha256:b33682f203818fcec713c1c7cbe0b01731c8b64991579ca95d1a6409823c652a \
```

It takes some time for the VMs to be provisioned, you can use the following command to wait until the VM workers reach ready state:
```
oc wait --for=condition=Ready --namespace $KUBEVIRT_CLUSTER_NAMESPACE vm --all --timeout=600s
```
Once the VM workers are ready, we can generate the guest cluster kubeconfig file which is required to retrieve guest cluster ingress NodePort.
```
hypershift create kubeconfig --name="$KUBEVIRT_CLUSTER_NAME" > "${KUBEVIRT_CLUSTER_NAME}-kubeconfig"
```
Export the ingress NodePort number to `$HTTPS_NODEPORT`. Please note that it might take some time for the certificate to be ready.
```
export HTTPS_NODEPORT=$(oc --kubeconfig "${KUBEVIRT_CLUSTER_NAME}-kubeconfig" get services -n openshift-ingress router-nodeport-default -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```
Once we have the NodePort number, we can create the ingress service and route for the guest cluster.

#### Ingress Service
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
#### Ingress Route
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
#### Verfiy the guest cluster is up and running
Once all the operators are deployed successfully, the status of the `Available` column should all be resolved to True and the Progress column should be changed to `Completed`.
```
[root@e24-h21-740xd ~]# oc get hc -A
NAMESPACE   NAME    VERSION   KUBECONFIG               PROGRESS    AVAILABLE   PROGRESSING   MESSAGE
clusters    kv-00   4.11.17   kv-00-admin-kubeconfig   Completed   True        False         The hosted control plane is available
```
## Summary
## Future Work
