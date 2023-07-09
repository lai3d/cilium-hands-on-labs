
```bash
root@server:~# kind create cluster --config cluster.yaml
Creating cluster "clab-bgp-cplane-devel" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-clab-bgp-cplane-devel"
You can now use your cluster with:

kubectl cluster-info --context kind-clab-bgp-cplane-devel

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
root@server:~# cat cluster.yaml
kind: Cluster
name: clab-bgp-cplane-devel
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: "172.0.0.2"
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5000"]
    endpoint = ["http://kind-registry:5000"]
root@server:~# kubectl get nodes
NAME                                  STATUS     ROLES           AGE     VERSION
clab-bgp-cplane-devel-control-plane   NotReady   control-plane   2m18s   v1.25.3
root@server:~# kubectl get nodes -o wide
NAME                                  STATUS     ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
clab-bgp-cplane-devel-control-plane   NotReady   control-plane   3m2s   v1.25.3   <none>        <none>        Ubuntu 22.04.1 LTS   5.15.0-1022-gcp   containerd://1.6.9
root@server:~# 
```


We can install containerlab with a single command:
```bash
bash -c "$(curl -sL https://get.containerlab.dev)" -- -v 0.31.1
```
Containerlab takes some of the best aspects of modern infrastructure deployment and applies them to networking:

You can use a declarative approach by writing network topologies in YAML
You can leverage open source and commercial routing platforms (in our case, the open source Free Range Routing platform (FRR))
You can use containers or virtual-machine to deploy your topology
You can leverage topology models that others have built (as users can share theirs on version control systems like GitHub).
In our case, we are using a topology file called topo.yaml. The topology file describes the networking platform to be deployed by containerlab.

```bash
root@server:~# bash -c "$(curl -sL https://get.containerlab.dev)" -- -v 0.31.1
Downloading https://github.com/srl-labs/containerlab/releases/download/v0.31.1/containerlab_0.31.1_linux_amd64.deb
Preparing to install containerlab 0.31.1 from package
Selecting previously unselected package containerlab.
(Reading database ... 64446 files and directories currently installed.)
Preparing to unpack .../containerlab_0.31.1_linux_amd64.deb ...
Unpacking containerlab (0.31.1) ...
Setting up containerlab (0.31.1) ...

                           _                   _       _     
                 _        (_)                 | |     | |    
 ____ ___  ____ | |_  ____ _ ____   ____  ____| | ____| | _  
/ ___) _ \|  _ \|  _)/ _  | |  _ \ / _  )/ ___) |/ _  | || \ 
( (__| |_|| | | | |_( ( | | | | | ( (/ /| |   | ( ( | | |_) )
\____)___/|_| |_|\___)_||_|_|_| |_|\____)_|   |_|\_||_|____/ 

    version: 0.31.1
     commit: 4fbe732d
       date: 2022-08-15T13:18:12Z
     source: https://github.com/srl-labs/containerlab
 rel. notes: https://containerlab.dev/rn/0.31/#0311
root@server:~# 
```

If you're curious, you can check out in details the containerlab topology we are deploying as part of the lab.
```bash
cat topo.yaml
```
The main thing to notice is that we are deploying 1 routing node: a single Top of Rack (ToR) router (tor). We are pre-configuring it at boot time with its IP and BGP configuration. At the end of the YAML file, you will also note we are establishing a virtual link between the Cilium node and the ToR router.

In the following tasks, we will configure Cilium to act as IP Address Management delegator for Kubernetes Services and we will run BGP and to establish BGP peering with the ToR device.

In this lab, we will use BGP to advertise both Pod CIDRs and LoadBalancer IPs allocated by Cilium LB IPAM.

```bash
root@server:~# cat topo.yaml 
name: bgp-cplane-devel
topology:
  kinds:
    linux:
      cmd: bash
  nodes:
    tor:
      kind: linux
      image: frrouting/frr:v8.2.2
      exec:
        # peer over this link with cilium
        - ip addr add 172.0.0.1/24 dev net0
        # Boiler plate to make FRR work
        - touch /etc/frr/vtysh.conf
        - sed -i -e 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
        - /usr/lib/frr/frrinit.sh start
        - >-
          vtysh -c 'conf t'
          -c 'debug bgp neighbor-events'
          -c 'debug bgp updates'
          -c 'debug bgp zebra'
          -c '! peer with Cilium'
          -c 'router bgp 65000'
          -c ' no bgp ebgp-requires-policy'    
          -c ' bgp router-id 172.0.0.1'
          -c ' neighbor 172.0.0.2 remote-as 65001'
          -c ' address-family ipv6 unicast'
          -c '  neighbor 172.0.0.2  activate'
          -c ' exit-address-family'
          -c '!'
    cilium:
      kind: linux
      image: nicolaka/netshoot:latest
      network-mode: container:control-plane
      exec:
        # peer over this link with tor
      - ip addr add 172.0.0.2/24 dev net0
  links:
    - endpoints: ["tor:net0", "cilium:net0"]
```

With the following command, we can deploy the topology previously described.
```bash
containerlab -t topo.yaml deploy
```
This typically only takes a few seconds to deploy.

In the next step, we will be deploying Cilium on the node.

```bash
root@server:~# containerlab -t topo.yaml deploy
INFO[0000] Containerlab v0.31.1 started                 
INFO[0000] Parsing & checking topology file: topo.yaml  
INFO[0000] Could not read docker config: open /root/.docker/config.json: no such file or directory 
INFO[0000] Pulling docker.io/frrouting/frr:v8.2.2 Docker image 
INFO[0005] Done pulling docker.io/frrouting/frr:v8.2.2  
INFO[0005] Could not read docker config: open /root/.docker/config.json: no such file or directory 
INFO[0005] Pulling docker.io/nicolaka/netshoot:latest Docker image 
INFO[0016] Done pulling docker.io/nicolaka/netshoot:latest 
INFO[0016] Creating lab directory: /root/clab-bgp-cplane-devel 
INFO[0016] Creating docker network: Name="clab", IPv4Subnet="172.20.20.0/24", IPv6Subnet="2001:172:20:20::/64", MTU="1500" 
INFO[0016] Creating container: "cilium"                 
INFO[0016] Creating container: "tor"                    
INFO[0019] Creating virtual wire: tor:net0 <--> cilium:net0 
INFO[0019] Adding containerlab host entries to /etc/hosts file 
INFO[0020] Executed command '/usr/lib/frr/frrinit.sh start' on clab-bgp-cplane-devel-tor. stdout:
Started watchfrr 
INFO[0020] 🎉 New containerlab version 0.42.0 is available! Release notes: https://containerlab.dev/rn/0.42/
Run 'containerlab version upgrade' to upgrade or go check other installation options at https://containerlab.dev/install/ 
+---+------------------------------+--------------+--------------------------+-------+---------+----------------+----------------------+
| # |             Name             | Container ID |          Image           | Kind  |  State  |  IPv4 Address  |     IPv6 Address     |
+---+------------------------------+--------------+--------------------------+-------+---------+----------------+----------------------+
| 1 | clab-bgp-cplane-devel-cilium | 3ef85358237a | nicolaka/netshoot:latest | linux | running | N/A            | N/A                  |
| 2 | clab-bgp-cplane-devel-tor    | bcfbaa2edd33 | frrouting/frr:v8.2.2     | linux | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
+---+------------------------------+--------------+--------------------------+-------+---------+----------------+----------------------+
```

The cilium CLI tool is provided in this environment to install and check the status of Cilium in the cluster.

Let's start by installing Cilium on the Kind cluster, with BGP enabled.

Use the following command:
```bash
cilium install --version=1.13.0-rc4 \
    --helm-set ipam.mode=kubernetes \
    --helm-set tunnel=disabled \
    --helm-set ipv4NativeRoutingCIDR="10.0.0.0/8" \
    --helm-set bgpControlPlane.enabled=true \
    --helm-set k8s.requireIPv4PodCIDR=true
```
Run cilium status to verify the health of Cilium. The status of Cilium and Operator should be both OK:

cilium status
Let's verify that BGP has been successfully enabled by checking the Cilium configuration:
```bash
cilium config view | grep enable-bgp
```
Next, we are going to learn about the LB-IPAM features.

```bash
root@server:~# cilium install --version=1.13.0-rc4 \
    --helm-set ipam.mode=kubernetes \
    --helm-set tunnel=disabled \
    --helm-set ipv4NativeRoutingCIDR="10.0.0.0/8" \
    --helm-set bgpControlPlane.enabled=true \
    --helm-set k8s.requireIPv4PodCIDR=true
🔮 Auto-detected Kubernetes kind: kind
✨ Running "kind" validation checks
✅ Detected kind version "0.17.0"
ℹ️  Using Cilium version 1.13.0-rc4
🔮 Auto-detected cluster name: kind-clab-bgp-cplane-devel
🔮 Auto-detected datapath mode: tunnel
🔮 Auto-detected kube-proxy has been installed
ℹ️  helm template --namespace kube-system cilium cilium/cilium --version 1.13.0-rc4 --set bgpControlPlane.enabled=true,cluster.id=0,cluster.name=kind-clab-bgp-cplane-devel,encryption.nodeEncryption=false,ipam.mode=kubernetes,ipv4NativeRoutingCIDR=10.0.0.0/8,k8s.requireIPv4PodCIDR=true,kubeProxyReplacement=disabled,operator.replicas=1,serviceAccounts.cilium.name=cilium,serviceAccounts.operator.name=cilium-operator,tunnel=disabled
ℹ️  Storing helm values file in kube-system/cilium-cli-helm-values Secret
🔑 Created CA in secret cilium-ca
🔑 Generating certificates for Hubble...
🚀 Creating Service accounts...
🚀 Creating Cluster roles...
🚀 Creating ConfigMap for Cilium version 1.13.0-rc4...
🚀 Creating Agent DaemonSet...
🚀 Creating Operator Deployment...
⌛ Waiting for Cilium to be installed and ready...
✅ Cilium was successfully installed! Run 'cilium status' to view installation health
root@server:~# cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         disabled
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/

DaemonSet         cilium             Desired: 1, Ready: 1/1, Available: 1/1
Deployment        cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
Containers:       cilium             Running: 1
                  cilium-operator    Running: 1
Cluster Pods:     3/3 managed by Cilium
Image versions    cilium             quay.io/cilium/cilium:v1.13.0-rc4@sha256:32acd47fd9bea9c0045222ba5d27f5fe9ad06dabd572a80b870b1f0e68c0e928: 1
                  cilium-operator    quay.io/cilium/operator-generic:v1.13.0-rc4@sha256:19f612d4f1052e26edf33e26f60d64d8fb6caed9f03692b85b429a4ef5d175b2: 1
root@server:~# cilium config view | grep enable-bgp
enable-bgp-control-plane                   true
```

# The Need for LoadBalancer IP Address Management
To allocate IP addresses for Kubernetes Services that are exposed outside of a cluster, you need a resource of the type LoadBalancer. When you use Kubernetes on a cloud provider, these resources are automatically managed for you and their IP and/or DNS are automatically allocated.

However if you run on a bare-metal cluster, you need another tool to allocate that address as Kubernetes doesn't natively support this function.

Typically you would have to install and use something like MetalLB for this purpose. Maintaining yet another networking tool can be cumbersome. In Cilium 1.13, you no longer need MetalLB for this use case: Cilium can allocate IP Addresses to Kubernetes LoadBalancer Service.

Let's have a look at this feature in more details.


Now that we have our virtual environment ready, let's start leveraging the new 1.13 feature "Load-Balancer IP Address Management" to allocate IP addresses to LoadBalancer Services. This will also be used in the next task to advertise LoadBalancer IPs out of the cluster over BGP.

Note this feature is enabled by default but dormant until the first IP Pool is added to the cluster.

```bash
root@server:~# cat pool.yaml
# No selector, match all
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "pool"
spec:
  cidrs:
  - cidr: "20.0.10.0/24"
root@server:~# 
```

# Cilium LB-IPAM

First things first: let's review a Cilium IPPool manifest. From this pool of IP addresses will be allocated IPs to Kubernetes Services of the type LoadBalancer.

LoadBalancer Services can be automatically created when exposing Services externally via the Ingress or Gateway API Resources but in this lab, we will be manually creating Services of the type LoadBalancer (LB-IPAM works with LoadBalancer Services, whether created manually or automatically).

Let's first review the IP Pool we will be using.
```bash
cat pool.yaml
```
IP addresses from the 20.0.10.0/24 range will be assigned to a LoadBalancer Service.

Let's review the service-blue Service. It is a simple Service of the type LoadBalancer that is labelled color:blue and is in the tenant-a namespace.
```bash
cat service-blue.yaml
```
Let's deploy it.
```bash
kubectl apply -f service-blue.yaml
```
Check that no IP address has been allocated yet (the External-IP should still be <pending>).
```bash
kubectl get svc/service-blue -n tenant-a
```
Let's now deploy the Cilium IP Pool:
```bash
kubectl apply -f pool.yaml
```
Check again the Service - you should now see an IP address from the 20.0.10.0/24 pool assigned in the field EXTERNAL-IP.
```bash
kubectl get svc/service-blue -n tenant-a
```

```bash
root@server:~# ll
total 120
drwx------  8 root root  4096 Jun 20 14:31 ./
drwxr-xr-x 19 root root  4096 Jun 20 14:17 ../
-rw-r--r--  1 root root    16 Nov 28  2022 .bash_aliases
-rw-r--r--  1 root root   523 Jun 20 14:33 .bash_history
-rw-r--r--  1 root root  1708 Jun 20 14:17 .bashrc
drwx------  4 root root  4096 Jun 20 14:29 .cache/
drwxr-xr-x  4 root root  4096 Nov 28  2022 .config/
drwxr-xr-x  3 root root  4096 Jun 20 14:20 .kube/
-rw-r--r--  1 root root   161 Jul  9  2019 .profile
drwx------  2 root root  4096 Jun 20 14:17 .ssh/
-rw-r--r--  1 root root  1167 Jun 20 14:27 .topo.yml.bak
-rw-------  1 root root 14583 Jun 20 14:17 .vimrc
-rw-r--r--  1 root root   215 Nov 28  2022 .wget-hsts
drwxr-xr-x  2 root root  4096 Jun 20 14:27 clab-bgp-cplane-devel/
-rw-r--r--  1 root root   426 Jun 20 14:17 cluster.yaml
-r-xr-xr-x  1 root root  3386 Jun 20 14:29 kind-shell-helpers.sh*
-r-xr-xr-x  1 root root   355 Jun 20 14:23 local_registry.sh*
-rw-r--r--  1 root root   222 Jun 20 14:31 pool-green.yaml
-rw-r--r--  1 root root   266 Jun 20 14:31 pool-primary.yaml
-rw-r--r--  1 root root   245 Jun 20 14:31 pool-yellow.yaml
-rw-r--r--  1 root root   154 Jun 20 14:31 pool.yaml
-rw-r--r--  1 root root   159 Jun 20 14:31 service-blue.yaml
-rw-r--r--  1 root root   161 Jun 20 14:31 service-green.yaml
-rw-r--r--  1 root root   157 Jun 20 14:31 service-red.yaml
-rw-r--r--  1 root root   257 Jun 20 14:31 service-yellow.yaml
drwx------  3 root root  4096 Nov 28  2022 snap/
-rw-r--r--  1 root root  1166 Jun 20 14:23 topo.yaml
root@server:~# cat service-blue.yaml
apiVersion: v1
kind: Service
metadata:
  name: service-blue
  namespace: tenant-a
  labels:
    color: blue
spec:
  type: LoadBalancer
  ports:
  - port: 1234
root@server:~# kubectl apply -f service-blue.yaml
service/service-blue created
root@server:~# kubectl get svc/service-blue -n tenant-a
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service-blue   LoadBalancer   10.96.45.196   <pending>     1234:30370/TCP   24s
root@server:~# kubectl apply -f pool.yaml
ciliumloadbalancerippool.cilium.io/pool created
root@server:~# kubectl get svc/service-blue -n tenant-a
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service-blue   LoadBalancer   10.96.45.196   20.0.10.80    1234:30370/TCP   52s
```


# Service Selectors

Operators may want to have applications and services assigned IP addresses from specific ranges. This might be useful to then enforce network security rules on an traditional border firewall.

For example, you may want Services tagged with a label environment:test or environment:prod to get IP addresses from different ranges.

In our example, we will keep using colors. Services tagged with blue, yellow or red will be allowed to pick up IP addresses from a primary colors IP pool.

We will be operating in the tenant-b namespace.

This would be done by using a Service Selector, with a regular expression such as the one defined in pool-primary.yaml.
```bash
cat pool-primary.yaml
```
The expression - {key: color, operator: In, values: [yellow, red, blue]} checks whether the Service requesting the IP has a label with the key color with a value of either yellow, red or blue.

Let's apply it (and delete the previously created one):
```bash
kubectl apply -f pool-primary.yaml
kubectl delete -f pool.yaml
```
Let's try this with a Service that hasn't got the right label and one with the right label.
```bash
kubectl apply -f service-red.yaml
kubectl apply -f service-green.yaml
```
Review their assigned IPs and labels with this command:
```bash
kubectl get svc -n tenant-b --show-labels
```
Unlike service-red, service-green does not get an IP address. It's simply because it doesn't match the regular expression defined in the previously defined pool (green is not one of the primary colors).

Let's now use an IP Pool that matches the color:green label:
```bash
cat pool-green.yaml
```
Let's now deploy it:
```bash
kubectl apply -f pool-green.yaml
```
This time, an IP address from the 40.0.10.0/24 should be assigned to service-green:
```bash
kubectl get svc/service-green -n tenant-b
```


```bash
root@server:~# cat pool-primary.yaml
# Expression selector
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
    name: "pool-primary"
spec:
  cidrs:
  - cidr: "60.0.10.0/24"
  serviceSelector:
    matchExpressions:
      - {key: color, operator: In, values: [yellow, red, blue]}
root@server:~# kubectl apply -f pool-primary.yaml
kubectl delete -f pool.yaml
ciliumloadbalancerippool.cilium.io/pool-primary created
ciliumloadbalancerippool.cilium.io "pool" deleted
root@server:~# kubectl apply -f service-red.yaml
kubectl apply -f service-green.yaml
service/service-red created
service/service-green created
root@server:~# kubectl get svc -n tenant-b --show-labels
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   LABELS
service-green   LoadBalancer   10.96.136.159   <pending>     1234:31297/TCP   6s    color=green
service-red     LoadBalancer   10.96.77.175    60.0.10.163   1234:31776/TCP   6s    color=red
root@server:~# cat service-red.yaml 
apiVersion: v1
kind: Service
metadata:
  name: service-red
  namespace: tenant-b
  labels:
    color: red
spec:
  type: LoadBalancer
  ports:
  - port: 1234
root@server:~# cat service-green.yaml 
apiVersion: v1
kind: Service
metadata:
  name: service-green
  namespace: tenant-b
  labels:
    color: green
spec:
  type: LoadBalancer
  ports:
  - port: 1234
root@server:~# cat pool-green.yaml
# # Second pool, label selector
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "pool-green"
spec:
  cidrs:
  - cidr: "40.0.10.0/24"
  serviceSelector:
    matchLabels:
      color: green
root@server:~# kubectl apply -f pool-green.yaml
ciliumloadbalancerippool.cilium.io/pool-green created
root@server:~# kubectl get svc/service-green -n tenant-b
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service-green   LoadBalancer   10.96.136.159   40.0.10.160   1234:31297/TCP   2m11s
```


# Requesting a specific IP

Users might want to request a specific set of IPs from a particular range. Cilium supports two methods to achieve this: the .spec.loadBalancerIP method (legacy, deprecated in K8S 1.24) and an annotation-based method.

The previous method has now been deprecated because it did not support dual-stack services (only a single IP could be requested). With the annotation, a list of requested (v4 or v6) IPs will be specified instead.

Review this yellow service. Note the annotation used to requested specific IPs. In this scenario, we are requesting 4 IP addresses:
```bash
cat service-yellow.yaml
```
We're also going to use another pool, this time, matching on the namespace (tenant-c in our example).
```bash
cat pool-yellow.yaml
```
Let's deploy them both:
```bash
kubectl apply -f service-yellow.yaml
kubectl apply -f pool-yellow.yaml
```
Observe the IP addresses that have been allocated:
```bash
kubectl get svc/service-yellow -n tenant-c
```
The output should be similar to this one:
```bash
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP               PORT(S)          AGE
service-yellow   LoadBalancer   10.96.238.88   50.0.10.100,60.0.10.100   1234:30158/TCP   33s
Note that the Service was allocated two of the requested IP addresses:

50.0.10.100 because it matches the namespace (tenant-c).
60.0.10.100 because it matches the primary colours labels.
Two other IP addresses were not assigned:

30.0.10.100 is not part of any defined pools.
40.0.10.100 is part of an existing pool but its serviceSelector doesn't match with the Service.
```
In the next task, we will use BGP to advertise the LoadBalancer Service IPs.



```bash
root@server:~# cat service-yellow.yaml
apiVersion: v1
kind: Service
metadata:
  name: service-yellow
  namespace: tenant-c
  labels:
    color: yellow
  annotations:
    "io.cilium/lb-ipam-ips": "30.0.10.100,40.0.10.100,50.0.10.100,60.0.10.100"
spec:
  type: LoadBalancer
  ports:
  - port: 1234

root@server:~# cat pool-yellow.yaml
# Namespace selector
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "pool-yellow"
spec:
  cidrs:
  - cidr: "50.0.10.0/24"
  serviceSelector:
    matchLabels:
      "io.kubernetes.service.namespace": "tenant-c"

root@server:~# kubectl apply -f service-yellow.yaml
kubectl apply -f pool-yellow.yaml
service/service-yellow created
ciliumloadbalancerippool.cilium.io/pool-yellow created

root@server:~# kubectl get svc/service-yellow -n tenant-c
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP               PORT(S)          AGE
service-yellow   LoadBalancer   10.96.94.147   50.0.10.100,60.0.10.100   1234:32019/TCP   12s
```



# BGP configuration



Let's first walk through the BGP Peering configuration.

Peering policies can be provisioned using simple Kubernetes CRDs, of the kind CiliumBGPPeeringPolicy.

You can use this command to review the peering policy or head out to the Editor tab:
```bash
cat $HOME/bgp/cilium-bgp-peering-policies.yaml
```
The key aspects of the policy are:

- the remote peer IP address (peerAddress) and AS Number (peerASN)
- your own local AS Number (localASN)
- Note the exportPodCIDR: true flag: it instructs Cilium to advertise to BGP neighbors the Pod CIDRs. This feature can be enabled or disabled, depending on your requirements.

- Note that BGP configuration on Cilium is label-based - only the Cilium-managed nodes with a matching label will deploy a virtual router for BGP peering purposes.

In our example, only nodes with the `kubernetes.io/hostname: clab-bgp-cplane-devel-control-plane` label will be running BGP.

This lab only has a single node and when you run this command filtering on this specific label, we should return the node (meaning that it is labelled as BGP router):
```bash
kubectl get nodes -l kubernetes.io/hostname=clab-bgp-cplane-devel-control-plane
```
For more details on the BGP configuration options, you can read up the official Cilium BGP documentations.

In the BGP Peering Policy provided, you will see some 5 commented lines (from 13 to 17): no need to uncomment them for now. We will do that in a later task.


```bash
root@server:~# cat $HOME/bgp/cilium-bgp-peering-policies.yaml
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeeringPolicy
metadata:
  name: tor
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/hostname: clab-bgp-cplane-devel-control-plane
  virtualRouters:
  - localASN: 65001
    exportPodCIDR: true
    # serviceSelector:
    #   matchLabels:
    #     color: yellow
    #   matchExpressions:
    #     - {key: io.kubernetes.service.namespace, operator: In, values: ["tenant-c"]}
    neighbors:
    - peerAddress: "172.0.0.1/32"
      peerASN: 65000
root@server:~# ll
total 124
drwx------  9 root root  4096 Jun 20 14:55 ./
drwxr-xr-x 19 root root  4096 Jun 20 14:17 ../
-rw-r--r--  1 root root    16 Nov 28  2022 .bash_aliases
-rw-r--r--  1 root root  1239 Jun 20 14:56 .bash_history
-rw-r--r--  1 root root  1708 Jun 20 14:17 .bashrc
drwx------  4 root root  4096 Jun 20 14:29 .cache/
drwxr-xr-x  4 root root  4096 Nov 28  2022 .config/
drwxr-xr-x  3 root root  4096 Jun 20 14:20 .kube/
-rw-r--r--  1 root root   161 Jul  9  2019 .profile
drwx------  2 root root  4096 Jun 20 14:17 .ssh/
-rw-r--r--  1 root root  1167 Jun 20 14:27 .topo.yml.bak
-rw-------  1 root root 14583 Jun 20 14:17 .vimrc
-rw-r--r--  1 root root   215 Nov 28  2022 .wget-hsts
drwxr-xr-x  3 root root  4096 Jun 20 14:55 bgp/
drwxr-xr-x  2 root root  4096 Jun 20 14:27 clab-bgp-cplane-devel/
-rw-r--r--  1 root root   426 Jun 20 14:17 cluster.yaml
-r-xr-xr-x  1 root root  3386 Jun 20 14:29 kind-shell-helpers.sh*
-r-xr-xr-x  1 root root   355 Jun 20 14:23 local_registry.sh*
-rw-r--r--  1 root root   222 Jun 20 14:31 pool-green.yaml
-rw-r--r--  1 root root   266 Jun 20 14:31 pool-primary.yaml
-rw-r--r--  1 root root   245 Jun 20 14:31 pool-yellow.yaml
-rw-r--r--  1 root root   154 Jun 20 14:31 pool.yaml
-rw-r--r--  1 root root   159 Jun 20 14:31 service-blue.yaml
-rw-r--r--  1 root root   161 Jun 20 14:31 service-green.yaml
-rw-r--r--  1 root root   157 Jun 20 14:31 service-red.yaml
-rw-r--r--  1 root root   257 Jun 20 14:31 service-yellow.yaml
drwx------  3 root root  4096 Nov 28  2022 snap/
-rw-r--r--  1 root root  1166 Jun 20 14:23 topo.yaml
root@server:~# ll bgp
total 16
drwxr-xr-x 3 root root 4096 Jun 20 14:55 ./
drwx------ 9 root root 4096 Jun 20 14:55 ../
-rw-r--r-- 1 root root  506 Jun 20 14:55 cilium-bgp-peering-policies.yaml
drwxr-xr-x 2 root root 4096 Jun 20 14:55 solved/
root@server:~# kubectl get nodes -l kubernetes.io/hostname=clab-bgp-cplane-devel-control-plane
NAME                                  STATUS   ROLES           AGE   VERSION
clab-bgp-cplane-devel-control-plane   Ready    control-plane   42m   v1.25.3
root@server:~# k get nodes
NAME                                  STATUS   ROLES           AGE   VERSION
clab-bgp-cplane-devel-control-plane   Ready    control-plane   43m   v1.25.3
```


# Deploy the BGP Peering Policies CRD configuration


Let's deploy the BGP peering policy.
```bash
kubectl apply -f $HOME/bgp/cilium-bgp-peering-policies.yaml
```

```bash
root@server:~# kubectl apply -f $HOME/bgp/cilium-bgp-peering-policies.yaml
ciliumbgppeeringpolicy.cilium.io/tor created
```


# Verify successful BGP peering


Now that we have set up our BGP peering, the peering session between the Cilium node and the Top of Rack switch should be established successfully.

Let's verify that the session has been established and that a route has been learned successfully (it might take a few seconds for the sessions to come up).

Run the following (see the next section for more information about this command).
```bash
docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4 summary wide'
```
It might take 10-15 seconds for the BGP session to come up but you should eventually see that the session between tor and the cilium node is Up (you will see how long the session has been up on the Up/Down column).

Verify with the following command:
```bash
docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4'
```
You should see the Pod CIDR 10.244.0.0/24 has been learned by our BGP peer.


```bash
root@server:~# docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4 summary wide'

IPv4 Unicast Summary (VRF default):
BGP router identifier 172.0.0.1, local AS number 65000 vrf-id 0
BGP table version 1
RIB entries 1, using 184 bytes of memory
Peers 1, using 716 KiB of memory

Neighbor        V         AS    LocalAS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
172.0.0.2       4      65001      65000         4         4        0    0    0 00:00:43            1        1 N/A

Total number of neighbors 1

root@server:~# docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4'
BGP table version is 1, local router ID is 172.0.0.1, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.244.0.0/24    172.0.0.2                              0 65001 i

Displayed  1 routes and 1 total paths
```


# BGP command explanation (optional)


If you are already familiar with BGP, then you can skip this part. If not, this section is a brief explanation of the BGP commands used in this task.

Let's explain briefly this command.

docker exec -it lets us enter the tor shell. As mentioned earlier, tor is based on the open-source Free Range Routing platform (FRR).
vtysh is the integrated shell on FRR devices.
show bgp ipv4 lets us check the BGP status.
Once you run this command, you will an output such as:
```bash
IPv4 Unicast Summary (VRF default):
BGP router identifier 172.0.0.1, local AS number 65000 vrf-id 0
BGP table version 1
RIB entries 1, using 184 bytes of memory
Peers 1, using 716 KiB of memory

Neighbor        V         AS    LocalAS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
172.0.0.2       4      65001      65000         5         5        0    0    0 00:01:11            1        1 N/A

Total number of neighbors 1
```
If you're familiar with using BGP on traditional CLIs such as Cisco IOS, this will look familiar. If not, let's go through some of the key outputs of the command above.

This commands provides information about the BGP status on tor. It shows tor's local AS number (65000), the remote AS number of the routers it is peering with (65000 for tor and 65001 for cilium).

It also shows, in the Up/Down column where the session is established (if that's the case, it will show for how long the session has been up - in our case, it's been up for 00:01:11).




# Advertise LoadBalancer Service routes


So far, we have only advertised the Pod CIDRs. Many users might actually prefer to only advertise the LoadBalancer Service IPs as these are designed to be externally exposed to clients.

With Cilium 1.13, you can advertise:

PodCIDRs,
LoadBalancer Service IPs
both,
neither.
Let's now advertise our LoadBalancer Service IPs to our neighbor.

Go to the Editor tab and uncomment the 5 lines from 13 to 17 of cilium-bgp-peering-policies.yaml.

Make sure you keep the indentation right (if you have any issues when applying the policy, just checkout the YAML file in the /root/bgp/solved/ folder to see what the right format is).

These 5 lines enable us to advertise LoadBalancer Service IP address of the Services with the color:yellow label and in the tenant-c namespace.

Make sure you save the file before heading back to Terminal and applying it:
```bash
kubectl apply -f /root/bgp/cilium-bgp-peering-policies.yaml
```
Let's verify that more routes ave been advertised.
```bash
docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4 summary wide'
```
The number of received BGP routes should have increased from 1 to 3.

Verify with the following command:
```bash
docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4'
```
You should still see the Pod CIDR 10.244.0.0/24 but also the LoadBalancer Service IPs that match the filters defined in the Service Selector (and that had specifically been requested from the IP Pools).

Here is the output you should expect to see:
```bash
BGP table version is 3, local router ID is 172.0.0.1, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.244.0.0/24    172.0.0.2                              0 65001 i
*> 50.0.10.100/32   172.0.0.2                              0 65001 i
*> 60.0.10.100/32   172.0.0.2                              0 65001 i

Displayed  3 routes and 3 total paths
```
Great job - you have successfully completed this lab and now understand how you can use BGP on Cilium to easily connect your Kubernetes clusters to your DC network.

Let's complete a short final quiz and an exam to validate our learnings.


```bash
root@server:~# kubectl apply -f /root/bgp/cilium-bgp-peering-policies.yaml
ciliumbgppeeringpolicy.cilium.io/tor configured
```
```bash
root@server:~# docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4 summary wide'
```
```
IPv4 Unicast Summary (VRF default):
BGP router identifier 172.0.0.1, local AS number 65000 vrf-id 0
BGP table version 3
RIB entries 5, using 920 bytes of memory
Peers 1, using 716 KiB of memory

Neighbor        V         AS    LocalAS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
172.0.0.2       4      65001      65000        18        17        0    0    0 00:06:49            3        3 N/A

Total number of neighbors 1
```
```bash
root@server:~# docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4'
```
```
BGP table version is 3, local router ID is 172.0.0.1, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.244.0.0/24    172.0.0.2                              0 65001 i
*> 50.0.10.100/32   172.0.0.2                              0 65001 i
*> 60.0.10.100/32   172.0.0.2                              0 65001 i

Displayed  3 routes and 3 total paths
```


# 🥋 Exam Challenge
The exam challenge starts where we left off in the previous task.

For this practical exam, you will need to advertise the service-green ExternalIP address to your neighbour.

Only the Service Green IP address is to be advertised.
You will need to edit and re-apply the existing BGP Peering Policy correctly.
There are many ways to select the service-green using a serviceSelector. The validation script will only verify if the service IP is advertised, not how you implemented it.
Make sure the Pod CIDR is not advertised.
The validation script will check whether the IP address received is from the 40.0.10.0/24 range.
You can use the commands you used previously to validate BGP:
```bash
docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4 summary wide'
docker exec -it clab-bgp-cplane-devel-tor vtysh -c 'show bgp ipv4'
```
ⓘ Notes:

The LB-IPAM documentation for Cilium can be found on https://docs.cilium.io/en/latest/network/lb-ipam/
The BGP documentation for Cilium can be found on https://docs.cilium.io/en/latest/network/bgp-control-plane/
Good luck!


