A close look at k8s networking model
===========================================================
## Last Updated
**Last Updated:** 2024-06-05 15:57 PM

## Introduction
This document covers the detailed tracing of how k8s networking model works in a single node openshift cluster environment.
## Cluster environment
We have a single node openshift cluster running within a QEMU/KVM VM. 
```
oc get nodes
NAME                STATUS   ROLES                         AGE   VERSION
52-54-00-a7-93-9c   Ready    control-plane,master,worker   27h   v1.28.9+8ca71f7

```
### VM config
The VM is called sno, assigned with an IP address 192.168.122.237/24
```
sudo virsh list
 Id   Name   State
----------------------
 4    sno    running
```

```
sudo virsh domifaddr sno
 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet3      52:54:00:a7:93:9c    ipv4         192.168.122.237/24

```
### Host config
virbr0 which is a virtual bridge interface created by libvirt for NAT(Network Address Translation).
```
8: virbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:62:66:de brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
```
### HTTP service
A sample web application is deployed 
```
oc get all -n demo-sno
NAME                                READY   STATUS      RESTARTS   AGE
pod/httpd-ex-git-1-build            0/1     Completed   0          26h
pod/httpd-ex-git-769765ffd5-frlp8   1/1     Running     0          26h

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
service/httpd-ex-git   ClusterIP   172.30.95.17   <none>        8080/TCP,8443/TCP   26h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/httpd-ex-git   1/1     1            1           26h

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/httpd-ex-git-769765ffd5   1         1         1       26h

NAME                                          TYPE     FROM   LATEST
buildconfig.build.openshift.io/httpd-ex-git   Source   Git    1

NAME                                      TYPE     FROM          STATUS     STARTED        DURATION
build.build.openshift.io/httpd-ex-git-1   Source   Git@6dd8df4   Complete   26 hours ago   32s

NAME                                          IMAGE REPOSITORY                                                         TAGS     UPDATED
imagestream.image.openshift.io/httpd-ex-git   image-registry.openshift-image-registry.svc:5000/demo-sno/httpd-ex-git   latest   26 hours ago

NAME                                    HOST/PORT                                    PATH   SERVICES       PORT       TERMINATION     WILDCARD
route.route.openshift.io/httpd-ex-git   httpd-ex-git-demo-sno.apps.sno.example.com          httpd-ex-git   8080-tcp   edge/Redirect   None
```
All the urls are pointing the a same IP address in /etc/hosts file.
```
# sno
192.168.122.237 console-openshift-console.apps.sno.example.com
192.168.122.237 api.sno.example.com
192.168.122.237 oauth-openshift.apps.sno.example.com
192.168.122.237 downloads-openshift-console.apps.sno.example.com
192.168.122.237 canary-openshift-ingress-canary.apps.sno.example.com
192.168.122.237 httpd-ex-git-demo-sno.apps.sno.example.com
```
```
getent hosts httpd-ex-git-demo-sno.apps.sno.example.com
192.168.122.237 httpd-ex-git-demo-sno.apps.sno.example.com
```
curling the service endpoint
```
curl -L -k httpd-ex-git-demo-sno.apps.sno.example.com
sno http test page
```
We have many URLs pointing to the same IP address, so how does the incoming/outgoing request know where the packets should be sent to? NAT is not applicable here since all URLs sharing the same IP address on port 80. By looking inside of the SNO node (VM), haproxy is listening all avaialble IPv4 address on port 80. 

```
netstat -lntp | grep :80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1371721/haproxy      │tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1371721/haproxy 
```
haproxy is also listening on port 443 which is the standard port for HTTPS.
```
netstat -lntp | grep haproxy
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      1371721/haproxy     
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1371721/haproxy
```
haproxy is able to forward the request using layer 7 load balancing to the corresponding service, in our case, httpd-ex-git under demo-sno namespace.
```
oc get route -n demo-sno
NAME           HOST/PORT                                    PATH   SERVICES       PORT       TERMINATION     WILDCARD
httpd-ex-git   httpd-ex-git-demo-sno.apps.sno.example.com          httpd-ex-git   8080-tcp   edge/Redirect   None
```

This http service is listening on TCP port 8080/8443 at address 172.30.95.17.
```
oc get service -n demo-sno
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
httpd-ex-git   ClusterIP   172.30.95.17   <none>        8080/TCP,8443/TCP   45h
```

Let's take a closer look at this service:
```
oc describe service httpd-ex-git -n demo-sno
Name:              httpd-ex-git
Namespace:         demo-sno
Labels:            app=httpd-ex-git
                   app.kubernetes.io/component=httpd-ex-git
                   app.kubernetes.io/instance=httpd-ex-git
                   app.kubernetes.io/name=httpd-ex-git
                   app.kubernetes.io/part-of=httpd-ex-git-app
                   app.openshift.io/runtime=httpd
                   app.openshift.io/runtime-version=latest
Annotations:       app.openshift.io/vcs-ref: 
                   app.openshift.io/vcs-uri: https://github.com/sclorg/httpd-ex.git
                   openshift.io/generated-by: OpenShiftWebConsole
Selector:          app=httpd-ex-git,deployment=httpd-ex-git
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                172.30.95.17
IPs:               172.30.95.17
Port:              8080-tcp  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.128.0.81:8080
Port:              8443-tcp  8443/TCP
TargetPort:        8443/TCP
Endpoints:         10.128.0.81:8443
Session Affinity:  None
Events:            <none>
```
This yaml tells us the service port to pod port mapping which are 8080 -> 8080 and 8443 -> 8443. The end point which is the pod IP is assigned as 10.128.0.81. Here is when OVN takes into play, making all the port mapping and request forwarding possible.

Let's take a look at OVN pods.
```
oc get pod -n openshift-ovn-kubernetes
NAME                                     READY   STATUS    RESTARTS   AGE
ovnkube-control-plane-78b4b5979b-9lxlw   2/2     Running   2          2d2h
ovnkube-node-j48k2                       8/8     Running   8          2d2h
```

By looking at the load balancing list, we can see that service IP 172.30.95.17 is mapped to 10.128.0.81 on port 8080 and 8443. Since our demo project has only pod pod deployment, we are only seeing the loadbalaner mapping to a single IP address.
```
oc exec -n openshift-ovn-kubernetes ovnkube-node-j48k2 -- ovn-nbctl lb-list | grep 172.30.95.17
Defaulted container "ovn-controller" out of: ovn-controller, ovn-acl-logging, kube-rbac-proxy-node, kube-rbac-proxy-ovn-metrics, northd, nbdb, sbdb, ovnkube-controller, kubecfg-setup (init)
8a65e790-0399-4f0f-8020-7dc4e57d371d    Service_demo-sno    tcp        172.30.95.17:8080       10.128.0.81:8080
                                                            tcp        172.30.95.17:8443       10.128.0.81:8443
```
Let's scale up our pod count to 2 and see how it looks like:

```
oc scale --replicas=2 deployment.apps/httpd-ex-git -n demo-sno
deployment.apps/httpd-ex-git scaled
```
We are have two replicas.
```
oc get pod -n demo-sno
NAME                            READY   STATUS      RESTARTS   AGE
httpd-ex-git-1-build            0/1     Completed   0          46h
httpd-ex-git-769765ffd5-frlp8   1/1     Running     0          46h
httpd-ex-git-769765ffd5-sb5zs   1/1     Running     0          38s
```
Without suprise, OVN is now load banlancing the requests on two pods with two different IP addresses.
```
oc exec -n openshift-ovn-kubernetes ovnkube-node-j48k2 -- ovn-nbctl lb-list | grep 172.30.95.17
Defaulted container "ovn-controller" out of: ovn-controller, ovn-acl-logging, kube-rbac-proxy-node, kube-rbac-proxy-ovn-metrics, northd, nbdb, sbdb, ovnkube-controller, kubecfg-setup (init)
8a65e790-0399-4f0f-8020-7dc4e57d371d    Service_demo-sno    tcp        172.30.95.17:8080       10.128.0.81:8080,10.128.1.99:8080
                                                            tcp        172.30.95.17:8443       10.128.0.81:8443,10.128.1.99:8443
```
Let's use tcpdump to actually trace the packet flow and to verify our understanding. Let's get into the node and start a tcpdump session. You may notice that tcpdump does come with coreos by default, however, we can use this handy tool called toolbox which have packed commonly used tools for debugging including dnf. 

before that, let's scale down the deployment to 1, so our tcpdump session will be more deterministic. 

```
oc scale --replicas=1 deployment.apps/httpd-ex-git -n demo-sno
deployment.apps/httpd-ex-git scaled
```

```
oc get pod -n demo-sno
NAME                            READY   STATUS      RESTARTS   AGE
httpd-ex-git-1-build            0/1     Completed   0          47h
httpd-ex-git-769765ffd5-frlp8   1/1     Running     0          47h
```
We can find out the actual container id running on the host using crictl tool which requires us to use the host binary by doing chroot /host

```
sh-5.1# crictl pods --namespace demo-sno --name httpd-ex-git-769765ffd5-frlp8 -q
6a0f2d307f4fa4af097705a652a3abf5ae9042d6209bf533e4d73d5d66610475
```
The pod ID we are interested is 6a0f2d307f4fa, we can inspect its content by:

```
crictl inspectp 6a0f2d307f4fa4af097705a652a3abf5ae9042d6209bf533e4d73d5d66610475 | grep -A3 network
    "network": {
      "additionalIps": [],
      "ip": "10.128.0.81"
    },
--
          "network": "POD",
          "pid": "CONTAINER",
          "targetId": "",
          "usernsOptions": null
--
            "type": "network",
            "path": "/var/run/netns/6a52c837-2ff1-4832-aa01-055d41d95b3d"
          },
          {
```
We are only interested in the network namespace so later we can enter that namespace find the network interface of that particular container.
