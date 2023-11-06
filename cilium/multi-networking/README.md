# Isovalent Enterprise for Cilium: Cilium Multi-Networking

Kubernetes is built on the premise that a Pod should belong to a single network.

While this approach may work for the majority of use cases, enterprise and telco often require a more sophisticated and flexible networking model.

There are many use cases where a Pod may require attachments to multiple networks with different properties via different interfaces.

With Cilium Multi-Networking, you can connect your Pod to multiple networks, without having to compromise on security and observability.

Start this lab to learn more.



🥈 Isovalent Enterprise for Cilium: Multi-Networking
🏛️ The Lab Environment

🚀 Multi Networks for Multiple Interfaces!

🚀 Let's deploy our demo app

🛡️ Enforce and test network policy

🔭 Observing traffic across multiple interfaces

❓ Multi Networking Quiz

🥋 Exam Challenge


⎈ Inflexible Networking
Kubernetes is built on the premise that a Pod should belong to a single network.

While this approach may work for the majority of use cases, enterprise and telco often require a more sophisticated and flexible networking model.

There are many use cases where a Pod may require attachments to multiple networks with different properties via different interfaces.


Progress
🔀 Multi Networking for Kubernetes
Isovalent Enterprise for Cilium 1.14 introduces support for Multi Networking for Kubernetes.

This is tremendously useful for a variety of use cases:

Network Segmentation:
Connecting Pods with multiple network interfaces can be used to segment network traffic. For example, you can have one interface for internal connectivity over a private network and another for external connectivity to the Internet.

Multi-Tenancy:
In a multi-tenant Kubernetes cluster, you can use Multi Networking alongside Cilium Network Policies to isolate network traffic between tenants by assigning different interfaces to different tenants or namespaces.

Service Chaining:
Service chaining is a network function virtualization (NFV) use case where multiple networking functions or services are applied to traffic as it flows to and from a Pod. Multi-Networking can help set up the necessary network interfaces for these services.

IoT and Edge Computing:
For IoT and edge computing scenarios, Multi-Networking can be used alongside Cilium Network Policies to impose network isolation on multi-tenant edge devices.


👮 Network Policies Support
Network Policies with Multi-Networking is supported from Day 1.

Each distinct network interface has its own identity and is treated as its own endpoint. As you will see in the subsequent lab, you can apply different policies to your different network interfaces.

You will try this in a later task in this lab.


🪪 Observability Support
One of the challenges with alternative multi-networking solutions is that traffic observability was limited.

Cilium Multi Networking supports native observability, with Hubble. You will explore this in a task later in the lab.

⚔️ Challenges & Quizzes
Most challenges of this lab will be followed by a short quiz, and the last challenge will test your understanding of the lab.

### The Kind Cluster

Let's have a look at this lab's environment.

We are running a Kind Kubernetes cluster, and on top of that Cilium.

While you wait for the Cilium installation to finish, let's have a look at the Kind configuration:

cat /etc/kind/${KIND_CONFIG}.yaml


### Nodes

In the nodes section, you can see that the cluster consists of three nodes:

1 control-plane node running the Kubernetes control plane and etcd
2 worker nodes to deploy the applications
We are exposing ports on the control plane node so we can access them from the terminal:

port 31234, used to access Hubble Relay (from the CLI)
port 31235, used to access the Hubble UI
Note that since the standard port for Hubble Relay is 4245, we have exported the HUBBLE_SERVER variable in the current shell to use the 31234 port instead. Verify its value with the following command:

echo $HUBBLE_SERVER

### Networking

In the networking section of the Kind configuration file, the default CNI has been disabled so the cluster won't have any Pod network when it starts. Instead, Cilium was deployed to the cluster to provide this functionality.

To see if the Kind cluster is ready, verify that the cluster is properly running by listing its nodes:

kubectl get nodes
You should see the three nodes appear. If not, the worker nodes might still be joining the cluster.


### Cilium and Hubble status

Let's also check if all Cilium components have been properly deployed. Note that it might take a few seconds to display the results!

cilium status --wait
If all is well, the Cilium, Operator and Hubble Relay lines should indicate OK.

You can also verify that you can properly connect to Hubble relay (using port 31234 in our lab) with:

hubble status
and that all nodes are properly managed in Hubble:

hubble list nodes
Finally, let's verify that Cilium has been enabled with the Multi-Networking feature.

The following command should return a true.

cilium config view | grep enable-multi-network
Now that we have a working Kind cluster, with Cilium enabled with the multi-networking option on, let's prepare the multi-networks before deploying multi-connected Pods.


### Default Pool and Network

To connect a Pod to multiple networks, we will first need multiple IP pools from which the Pod's network interfaces can get their IP address.

During the lab boot-up, we used a config option asking Cilium to automatically create a default pool and network.

Let's verify a default IP pool was automatically created:

kubectl get ciliumpodippools.cilium.io default -o json | jq .spec
Expect an output such as:

{
  "ipv4": {
    "cidrs": [
      "10.10.0.0/16"
    ],
    "maskSize": 24
  }
}
10.10.0.0/16 is the full CIDR allocated to this pool and smaller /24 subnets will be allocated to the nodes when required.

Let's verify a default network was automatically created:

kubectl get isovalentpodnetworks default -o json | jq .spec
Expect an output such as:

{
  "ipam": {
    "mode": "multi-pool",
    "pool": {
      "name": "default"
    }
  }
}
The default network leverages the default IP pool.

Let's also verify that smaller /24 IP pools have been carved out and assigned to both kind-worker and kind-worker2.

Note
Multi Pool is an evolution of the Cluster Scope IPAM model and provides even more flexibility and granularity on how IP addresses are allocated. To learn more, check the docs.

kubectl get ciliumnode kind-worker2 -o json | jq ".spec.ipam.pools.allocated"
kubectl get ciliumnode kind-worker -o json | jq ".spec.ipam.pools.allocated"
Expect an output such as:

[
  {
    "cidrs": [
      "10.10.1.0/24"
    ],
    "pool": "default"
  }
]
[
  {
    "cidrs": [
      "10.10.2.0/24"
    ],
    "pool": "default"
  }
]
Our default network and default pool of IPs will be used by workloads that don't specify the annotation required to join a secondary network.


### Secondary IP Pool and IP Network

Let's now deploy a secondary IP Pool.

First, review the IP pool:

cat jupiter-ip-pool.yaml
Similarly to the default pool, a large CIDR block (192.168.16.0/20) was allocated to the pool and smaller blocks (based on a /27 mask) will be allocated to the nodes when Pods are deployed on this network.

Deploy it:

kubectl apply -f jupiter-ip-pool.yaml
Let's now deploy a new network (based on the IsovalentPodNetwork specification) that will use the newly created pool.

First, review the network configuration.:

cat jupiterPodNetwork.yaml
The new IsovalentPodNetwork named jupiter uses the jupiter-ip-pool Pod IP pool.

Note the routes section: it it used to steer traffic within the Pods towards the secondary network.

Let's now deploy it.

kubectl apply -f jupiterPodNetwork.yaml

### Disabling Masquerading

Finally, we also need to exclude the CIDR of the secondary network from masquerading.

IP Masquerading is a form of Source Network Address Translation (SNAT). When a Pod sends traffic to an external destination, Cilium replaces the source IP address (the internal Pod IP) with the IP address of the node.

Note
You can read more about Masquerading on the Cilium Docs page.

With multi-networking, we do not want IP masquerading to happen as we want the Pod's multiple IPs to be respected throughout the network.

Masquerading therefore needs to be disabled.

The default network is not masqueraded by default but we now need to do it for our secondary network.

To achieve that, we need to deploy a Config Map specifying the prefix where masquerading should not happen.

Here is the Config Map you will use:

cat nonMasq.yaml
As we enabled the ipMasqAgent.true option at boot time, Cilium will then check corresponding Config Maps and will disable masquerading based on the specific prefix.

Once you deploy the nonMasq.yaml, masquerading for the 192.168.0.0/16 CIDR will be disabled.

Let's deploy it:

kubectl apply -f nonMasq.yaml
Let's now proceed to the next task to deploy our multi-networked Pods!


### Console

```shell
root@server:~# kubectl get ciliumpodippools.cilium.io default -o json | jq .spec
{
  "ipv4": {
    "cidrs": [
      "10.10.0.0/16"
    ],
    "maskSize": 24
  }
}
root@server:~# kubectl get isovalentpodnetworks default -o json | jq .spec
{
  "ipam": {
    "mode": "multi-pool",
    "pool": {
      "name": "default"
    }
  }
}
root@server:~# kubectl get ciliumnode kind-worker2 -o json | jq ".spec.ipam.pools.allocated"
kubectl get ciliumnode kind-worker -o json | jq ".spec.ipam.pools.allocated"
[
  {
    "cidrs": [
      "10.10.2.0/24"
    ],
    "pool": "default"
  }
]
[
  {
    "cidrs": [
      "10.10.0.0/24"
    ],
    "pool": "default"
  }
]
root@server:~# cat jupiter-ip-pool.yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumPodIPPool
metadata:
  name: jupiter-ip-pool
spec:
  ipv4:
    cidrs:
    - 192.168.16.0/20
    maskSize: 27
root@server:~# kubectl apply -f jupiter-ip-pool.yaml
ciliumpodippool.cilium.io/jupiter-ip-pool created
root@server:~# cat jupiterPodNetwork.yaml
---
apiVersion: isovalent.com/v1alpha1
kind: IsovalentPodNetwork
metadata:
  name: jupiter
spec:
  ipam:
    mode: multi-pool
    pool:
      name: jupiter-ip-pool
  routes:
  - destination: 192.168.0.0/16
    gateway: 192.168.0.1
root@server:~# kubectl apply -f jupiterPodNetwork.yaml
isovalentpodnetwork.isovalent.com/jupiter created
root@server:~# cat nonMasq.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ip-masq-agent
  namespace: kube-system
data:
  config: |
    nonMasqueradeCIDRs:
    - 192.168.0.0/16
root@server:~# kubectl apply -f nonMasq.yaml
configmap/ip-masq-agent created
root@server:~# 
```

## 🚀 Multi Networking
In the following challenge, we will deploy a multi-networked client Pod for the lab and two servers on two different networks and that the client Pod can connect to both over the correct interfaces.


### Attach a Pod to Multiple Networks

Let's deploy these Pods (two server Pods and one multi-networked client Pod). We will review their config after:

kubectl apply -f server-default.yaml \
  -f server-jupiter.yaml \
  -f client.yaml
Let's now review their config.

cat server-default.yaml
cat server-jupiter.yaml
cat client.yaml
To create a Pod which is attached to additional networks, we'll use the network.v1alpha1.isovalent.com/pod-networks annotation.

This annotation consists of a comma-delimited list of IsovalentPodNetwork names.

For example, the annotation network.v1alpha1.isovalent.com/pod-networks: default,jupiter causes Cilium to attach the client pod to the default and jupiter networks.

Pod without the annotation will only be attached to the default network.

To recap:

Pod server-default will be attached only to the default network.
Pod server-jupiter will be attached only to the jupiter network.
Pod client wiil be attached to both networks.

### Verify the network connectivity

Let's now validate that the three Pods are connected to their respective networks and are able to connect to each other.

Verify that all three Pods are running:

kubectl get pods -o wide
Expect an output similar to this:

NAME             READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
client           1/1     Running   0          64m   10.10.2.232     kind-worker2   <none>           <none>
server-default   1/1     Running   0          64m   10.10.2.18      kind-worker2   <none>           <none>
server-jupiter   1/1     Running   0          64m   192.168.16.30   kind-worker2   <none>           <none>
It may take 20 seconds or so for the Pods to show as Running.

Note
From this output, you can only see the main IP address attached to client.

Next, let's have a look at the Cilium Endpoints.

Cilium refers to all application containers which share a common IP address as a single endpoint. Each endpoint is assigned a security identity.

Let's now verify that a CiliumEndpoint resource is present for each server pod and two CiliumEndpoint resources are present for the client pod:

kubectl get ciliumendpoints.cilium.io
Expect an output similar to this:

NAME             ENDPOINT ID   IDENTITY ID   INGRESS ENFORCEMENT   EGRESS ENFORCEMENT   VISIBILITY POLICY   ENDPOINT STATE   IPV4            IPV6
client           450           12551         <status disabled>     <status disabled>    <status disabled>   ready            10.10.2.232
client-cil1      3220          32044         <status disabled>     <status disabled>    <status disabled>   ready            192.168.16.18
server-default   689           12551         <status disabled>     <status disabled>    <status disabled>   ready            10.10.2.18
server-jupiter   129           12551         <status disabled>     <status disabled>    <status disabled>   ready            192.168.16.30
The client endpoint represents the primary Pod interface and the client-cil1 endpoint represents the secondary Pod interface of the client Pod. Both server Pods only have a single endpoint because they are each only attached to a single network.

Let's verify that a secondary network interface cil1 was created in the client Pod, connecting it to the jupiter network:

kubectl exec -it client -- ip addr show
Expect an output such as:

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 4e:20:7a:77:86:b5 brd ff:ff:ff:ff:ff:ff
    inet 10.10.2.232/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::4c20:7aff:fe77:86b5/64 scope link
       valid_lft forever preferred_lft forever
13: cil1@if14: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether a2:c8:af:18:aa:56 brd ff:ff:ff:ff:ff:ff
    inet 192.168.16.18/32 scope global cil1
       valid_lft forever preferred_lft forever
    inet6 fe80::a0c8:afff:fe18:aa56/64 scope link
       valid_lft forever preferred_lft forever
Note the cil1 IP (192.168.16.18 in the example above) matches the client-cil1 interface IP address in the Cilium endpoint list above.

Let's extract the IP address for server-default and server-jupiter and assign it to environment variables.

SERVER_DEFAULT_IP=$(kubectl get pod server-default -o jsonpath='{.status.podIP}')
SERVER_JUPITER_IP=$(kubectl get pod server-jupiter -o jsonpath='{.status.podIP}')
Let's double-check the variables:

echo "server-default IP: $SERVER_DEFAULT_IP"
echo "server-jupiter IP: $SERVER_JUPITER_IP"
Validate the connectivity in the default network (10.10.0.0/16) from the multi-networked client Pod to the server-default Pod using ping:

kubectl exec -it client -- ping -c 1 $SERVER_DEFAULT_IP
The ping should be successful.

Next, validate that the server is reachable by sending an HTTP GET request to the server's /client-ip endpoint. In its reply it will report the client's IP address:

kubectl exec -it client -- curl $SERVER_DEFAULT_IP/client-ip
The curl should be successful. Expect an output such as:

{
  "client-ip": "::ffff:10.10.2.232"
}
Similarly, validate connectivity in the jupiter network (192.168.16.0/20) from the multi-networked client pod to the server-jupiter pod:

kubectl exec -it client -- ping -c 1 $SERVER_JUPITER_IP
The ping should be successful.

Next, validate that the server is reachable by sending an HTTP GET request to the server's /client-ip endpoint. In its reply it will report the client's IP address:

kubectl exec -it client -- curl $SERVER_JUPITER_IP/client-ip
The curl should be successful. Expect an output such as:

{
  "client-ip": "::ffff:192.168.16.18"
}
Connectivity worked and, as you can see from the response in the packet, the packets sent from client to both server Pods were sent from a different IP. The source IP wasn't masqueraded and remained that of the original Pod and it originated from the Pod's correct network interface.


### What about network policies?

Ping and HTTP traffic tests were both successful.

But can we apply different network policies to these different network interfaces?

Let's find out in the next step.

### Console

```shell
root@server:~# kubectl apply -f server-default.yaml \
  -f server-jupiter.yaml \
  -f client.yaml
pod/server-default created
pod/server-jupiter created
pod/client created
root@server:~# cat server-default.yaml
cat server-jupiter.yaml
cat client.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: server-default
  labels:
    app: server-default
  annotations:
    network.v1alpha1.isovalent.com/pod-networks: default
spec:
  containers:
  - name: server-jupiter-container
    image: quay.io/cilium/json-mock:v1.3.5
---
apiVersion: v1
kind: Pod
metadata:
  name: server-jupiter
  labels:
    app: server-jupiter
  annotations:
    network.v1alpha1.isovalent.com/pod-networks: jupiter
spec:
  containers:
  - name: server-jupiter-container
    image: quay.io/cilium/json-mock:v1.3.5
---
apiVersion: v1
kind: Pod
metadata:
  name: client
  labels:
    app: client
  annotations:
    network.v1alpha1.isovalent.com/pod-networks: default,jupiter
spec:
  containers:
  - name: client-container
    image: quay.io/cilium/alpine-curl:v1.5.0
    command:
      - /bin/ash
      - -c
      - sleep 10000000
root@server:~# kubectl get pods -o wide
NAME             READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
client           1/1     Running   0          2m49s   10.10.2.66      kind-worker2   <none>           <none>
server-default   1/1     Running   0          2m49s   10.10.2.245     kind-worker2   <none>           <none>
server-jupiter   1/1     Running   0          2m49s   192.168.16.20   kind-worker2   <none>           <none>
root@server:~# kubectl get ciliumendpoints.cilium.io
NAME             ENDPOINT ID   IDENTITY ID   INGRESS ENFORCEMENT   EGRESS ENFORCEMENT   VISIBILITY POLICY   ENDPOINT STATE   IPV4            IPV6
client           1502          16946         <status disabled>     <status disabled>    <status disabled>   ready            10.10.2.66      
client-cil1      1786          39536         <status disabled>     <status disabled>    <status disabled>   ready            192.168.16.17   
server-default   3103          47491         <status disabled>     <status disabled>    <status disabled>   ready            10.10.2.245     
server-jupiter   567           21927         <status disabled>     <status disabled>    <status disabled>   ready            192.168.16.20   
root@server:~# kubectl exec -it client -- ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
9: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether de:ab:f2:6d:fc:f5 brd ff:ff:ff:ff:ff:ff
    inet 10.10.2.66/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::dcab:f2ff:fe6d:fcf5/64 scope link 
       valid_lft forever preferred_lft forever
11: cil1@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether fe:4b:77:d5:c5:75 brd ff:ff:ff:ff:ff:ff
    inet 192.168.16.17/32 scope global cil1
       valid_lft forever preferred_lft forever
    inet6 fe80::fc4b:77ff:fed5:c575/64 scope link 
       valid_lft forever preferred_lft forever
root@server:~# SERVER_DEFAULT_IP=$(kubectl get pod server-default -o jsonpath='{.status.podIP}')
SERVER_JUPITER_IP=$(kubectl get pod server-jupiter -o jsonpath='{.status.podIP}')
root@server:~# echo "server-default IP: $SERVER_DEFAULT_IP"
echo "server-jupiter IP: $SERVER_JUPITER_IP"
server-default IP: 10.10.2.245
server-jupiter IP: 192.168.16.20
root@server:~# kubectl exec -it client -- ping -c 1 $SERVER_DEFAULT_IP
PING 10.10.2.245 (10.10.2.245) 56(84) bytes of data.
64 bytes from 10.10.2.245: icmp_seq=1 ttl=63 time=0.181 ms

--- 10.10.2.245 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.181/0.181/0.181/0.000 ms
root@server:~# kubectl exec -it client -- curl $SERVER_DEFAULT_IP/client-ip
{
  "client-ip": "::ffff:10.10.2.66"
}root@server:~#kubectl exec -it client -- ping -c 1 $SERVER_JUPITER_IPP
PING 192.168.16.20 (192.168.16.20) 56(84) bytes of data.
64 bytes from 192.168.16.20: icmp_seq=1 ttl=63 time=0.187 ms

--- 192.168.16.20 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.187/0.187/0.187/0.000 ms
root@server:~# kubectl exec -it client -- curl $SERVER_JUPITER_IP/client-ip
{
  "client-ip": "::ffff:192.168.16.17"
}root@server:~# 
```

## 🧪 Testing Network Policy with Multi-Networking
A key feature of Cilium is the support for advanced Network Policies.

Let's demonstrate that Pods with multiple network interfaces can have different network policies applied to them.

### Introduction

Traditionally security enforcement architectures have been based on IP address filters. It's largely irrelevant in the cloud native world given the churn and the scale of micro-services.

In order to avoid these complications which can limit scalability and flexibility, Cilium entirely separates security from network addressing.

Instead, security is based on the identity of a pod, which is derived through labels.

Let's have a look at the identity ID of the client endpoint:

kubectl get cep client -o json | jq .status.identity
Expect an output such as:

{
 "id": 28348,
 "labels": [
   "k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default",
   "k8s:io.cilium.k8s.policy.cluster=default",
   "k8s:io.cilium.k8s.policy.serviceaccount=default",
   "k8s:io.kubernetes.pod.namespace=default"
 ]
}
id is the identity ID and the labels are the set of Pod labels associated to this identity.

Let's now have a look at the identiy ID of the client-cil1 endpoint - in other words, the secondary interface on the same client Pod.

kubectl get cep client-cil1 -o json | jq .status.identity
Expect an output such as:

{
  "id": 20285,
  "labels": [
    "cni:com.isovalent.v1alpha1.network.attachment=jupiter",
    "k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default",
    "k8s:io.cilium.k8s.policy.cluster=default",
    "k8s:io.cilium.k8s.policy.serviceaccount=default",
    "k8s:io.kubernetes.pod.namespace=default"
  ]
}
Both identity IDs are different, which means we can apply different network policies to it.

When using Multi Networking, every Cilium Endpoint associated with a non-default network has one additional label of the form cni:com.isovalent.v1alpha1.network.attachment=<network name> added by the Cilium CNI plugin.

This is how each network interface will have its own Cilium security identity, and it therefore allows us to write network policies that only apply to a specific network by selecting this extra label.

### Apply the policies

Let's first review our policies. You can also use the </> Editor tab to review them.

cat default-netpol.yaml
cat jupiter-netpol.yaml
The network policies are Layer 7 (L7) Cilium Network Policies. They provide a more granular model than the traditional Kubernetes Network Policies, which can only support L3/L4 constructs.

The default-netpol.yaml network policy allows traffic from the client Pod to server-default, only if that traffic is HTTP using method GET and to "/client-ip" path.

The jupiter-netpol.yaml network policy allows traffic from endpoints in the jupiter network to server-jupiter, only if that traffic is HTTP using method GET and to "/public" path.

Let's apply the policy and verify it:

kubectl apply -f jupiter-netpol.yaml
kubectl apply -f default-netpol.yaml
Did it work? The best way to find out is to replay the connection tests we used earlier!

### Test network policies

Let's extract the IPs of our server Pods again.

SERVER_DEFAULT_IP=$(kubectl get pod server-default -o jsonpath='{.status.podIP}')
SERVER_JUPITER_IP=$(kubectl get pod server-jupiter -o jsonpath='{.status.podIP}')
echo "server-default IP: $SERVER_DEFAULT_IP"
echo "server-jupiter IP: $SERVER_JUPITER_IP"
First, validate that the server is reachable by sending an HTTP GET request to the server's /client-ip endpoint. In its reply it will report the client's IP address:

kubectl exec -it client -- curl $SERVER_DEFAULT_IP/client-ip
It should still be successful. Expect an output such as:

{
  "client-ip": "::ffff:10.10.2.232"
}
Notice the IPv4's client IP address in the reply.

Now, validate that the connectivity in the default network (10.10.0.0/16) from the multi-networked client Pod to the server-default Pod using ping is blocked:

kubectl exec -it client -- ping -c 1  -W 1 $SERVER_DEFAULT_IP
The ping should not be successful.

Verify that access to any other HTTP path is also blocked:

kubectl exec -it client -- curl $SERVER_DEFAULT_IP/public
Expect Access denied in the output.

Similarly, validate connectivity in the jupiter network (192.168.16.0/20) from the multi-networked client Pod to the server-jupiter Pod:

kubectl exec -it client -- ping -c 1  -W 1 $SERVER_JUPITER_IP
The ping should not be successful.

Next, make a HTTP request to the /client-ip path:

kubectl exec -it client -- curl $SERVER_JUPITER_IP/client-ip
It should still not be successful and you should get an Access denied error again.

Finally, let's try a HTTP request to the /public path:

kubectl exec -it client -- curl $SERVER_JUPITER_IP/public
It should be successful. Expect this output:

[
  {
    "id": 1,
    "body": "public information"
  }
]
As you can see, we can apply different network policies to our network interfaces!

In the next challenge, we'll see how we can visualize these flows using the Hubble CLI !

### Console

```shell
root@server:~# kubectl get cep client -o json | jq .status.identity
{
  "id": 16946,
  "labels": [
    "k8s:app=client",
    "k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default",
    "k8s:io.cilium.k8s.policy.cluster=default",
    "k8s:io.cilium.k8s.policy.serviceaccount=default",
    "k8s:io.kubernetes.pod.namespace=default"
  ]
}
root@server:~# kubectl get cep client-cil1 -o json | jq .status.identity
{
  "id": 39536,
  "labels": [
    "cni:com.isovalent.v1alpha1.network.attachment=jupiter",
    "k8s:app=client",
    "k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=default",
    "k8s:io.cilium.k8s.policy.cluster=default",
    "k8s:io.cilium.k8s.policy.serviceaccount=default",
    "k8s:io.kubernetes.pod.namespace=default"
  ]
}
root@server:~# cat default-netpol.yaml
cat jupiter-netpol.yaml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: server-default-policy
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: server-default
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: client
      toPorts:
        - ports:
          - port: "80"
            protocol: TCP
          rules:
            http:
            - method: "GET"
              path: "/client-ip"
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: server-jupiter-policy
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: server-jupiter
  ingress:
    - fromEndpoints:
        - matchLabels:
            cni:com.isovalent.v1alpha1.network.attachment: jupiter
      toPorts:
        - ports:
          - port: "80"
            protocol: TCP
          rules:
            http:
            - method: "GET"
              path: "/public"
root@server:~# kubectl apply -f jupiter-netpol.yaml
kubectl apply -f default-netpol.yaml
ciliumnetworkpolicy.cilium.io/server-jupiter-policy created
ciliumnetworkpolicy.cilium.io/server-default-policy created
root@server:~# SERVER_DEFAULT_IP=$(kubectl get pod server-default -o jsonpath='{.status.podIP}')
SERVER_JUPITER_IP=$(kubectl get pod server-jupiter -o jsonpath='{.status.podIP}')
echo "server-default IP: $SERVER_DEFAULT_IP"
echo "server-jupiter IP: $SERVER_JUPITER_IP"
server-default IP: 10.10.2.245
server-jupiter IP: 192.168.16.20
root@server:~# kubectl exec -it client -- curl $SERVER_DEFAULT_IP/client-ip
{
  "client-ip": "::ffff:172.18.0.2"
}root@server:~#kubectl exec -it client -- ping -c 1  -W 1 $SERVER_DEFAULT_IPP
PING 10.10.2.245 (10.10.2.245) 56(84) bytes of data.

--- 10.10.2.245 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

command terminated with exit code 1
root@server:~# kubectl exec -it client -- curl $SERVER_DEFAULT_IP/public
Access denied
root@server:~# kubectl exec -it client -- ping -c 1  -W 1 $SERVER_JUPITER_IP
PING 192.168.16.20 (192.168.16.20) 56(84) bytes of data.

--- 192.168.16.20 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

command terminated with exit code 1
root@server:~# kubectl exec -it client -- curl $SERVER_JUPITER_IP/client-ip
Access denied
root@server:~# kubectl exec -it client -- curl $SERVER_JUPITER_IP/public
[
  {
    "id": 1,
    "body": "public information"
  }
]root@server:~# 
```

## 🔍 Visualizing dropped traffic
We have successfully applied new rules and verified that connections are now not allowed.

But that means traffic was dropped - but how do we verify that traffic was dropped?

## 🛰️ Hubble to the rescue
The Hubble Observability platform will offer insight into connection drops.

In this short challenge we will use the Hubble CLI to analyse and filter packet flows.


### Observing Flows

Using the hubble CLI, we can see all the requests from the client Pod - across both network interfaces it's connected to:

hubble observe  --pod client --protocol icmp
You should see an output such as:

Oct 12 17:03:57.440: default/client (ID:33188) -> default/server-default (ID:61802) to-stack FORWARDED (ICMPv4 EchoRequest)
Oct 12 17:03:57.440: default/client (ID:33188) <> default/server-default (ID:61802) policy-verdict:none INGRESS DENIED (ICMPv4 EchoRequest)
Oct 12 17:03:57.440: default/client (ID:33188) <> default/server-default (ID:61802) Policy denied DROPPED (ICMPv4 EchoRequest)
Oct 12 17:03:59.968: default/client (ID:59227) -> default/server-jupiter (ID:57192) to-stack FORWARDED (ICMPv4 EchoRequest)
Oct 12 17:03:59.968: default/client (ID:59227) <> default/server-jupiter (ID:57192) policy-verdict:none INGRESS DENIED (ICMPv4 EchoRequest)
Oct 12 17:03:59.968: default/client (ID:59227) <> default/server-jupiter (ID:57192) Policy denied DROPPED (ICMPv4 EchoRequest)
You will see flows marked as FORWARDED, DENIED or DROPPED.

You can filter on this criteria with the --verdict flag, for example executing:

hubble observe  --pod client --protocol icmp --last 50 --verdict=DROPPED
You should see an output such as:

Oct 13 09:22:41.317: default/client (ID:51632) <> default/server-default (ID:18899) policy-verdict:none INGRESS DENIED (ICMPv4 EchoRequest)
Oct 13 09:22:41.317: default/client (ID:51632) <> default/server-default (ID:18899) Policy denied DROPPED (ICMPv4 EchoRequest)
Oct 13 09:27:24.196: default/client (ID:64356) <> default/server-jupiter (ID:25697) policy-verdict:none INGRESS DENIED (ICMPv4 EchoRequest)
Oct 13 09:27:24.196: default/client (ID:64356) <> default/server-jupiter (ID:25697) Policy denied DROPPED (ICMPv4 EchoRequest)
If you want to see traffic coming from a particular interface, you can use the identity assigned to the endpoint of the interface. In the example above, 33188 is the identity ID of the client and 59227 is the identity of the secondary interface (client-cil1).

Let's extract the identity of the client and client-cli1 endpoints and assign them to environment variables:

CLIENT_ID=$(kubectl get cep client -o jsonpath='{.status.identity.id}')
CLIENT_CIL1_ID=$(kubectl get cep client-cil1 -o jsonpath='{.status.identity.id}')
echo $CLIENT_ID
echo $CLIENT_CIL1_ID
Filter to the interface of your choice with:

hubble observe  --pod client --protocol icmp --last 50 --verdict=DROPPED --identity $CLIENT_ID
You can also filter based on HTTP attributes, like the HTTP status code. You can find all successful responses with this command:

hubble observe --http-status 200
You should be able to view the successful responses from the previous challenge:

Oct 16 07:45:14.344: default/client-cil1:37106 (ID:37089) <- default/server-jupiter:80 (ID:10676) http-response FORWARDED (HTTP/1.1 200 7ms (GET http://192.168.16.11/public))
Oct 16 07:46:35.803: default/client:53276 (ID:29503) <- default/server-default:80 (ID:10031) http-response FORWARDED (HTTP/1.1 200 3ms (GET http://10.10.3.168/client-ip))
As you can see, even when using Cilium Multi-Networking, we don't lose the observability insight that Hubble gives us.

In the next task, you will take a short quiz before attempting the final exam challenge!

### Console

```shell
root@server:~# hubble observe  --pod client --protocol icmp
Nov  6 02:52:28.104: default/client (ID:16946) -> default/server-default (ID:47491) to-stack FORWARDED (ICMPv4 EchoRequest)
Nov  6 02:52:28.104: default/client (ID:16946) -> default/server-default (ID:47491) to-endpoint FORWARDED (ICMPv4 EchoRequest)
Nov  6 02:52:28.104: default/client (ID:16946) <- default/server-default (ID:47491) to-stack FORWARDED (ICMPv4 EchoReply)
Nov  6 02:52:28.104: default/client (ID:16946) <- default/server-default (ID:47491) to-endpoint FORWARDED (ICMPv4 EchoReply)
Nov  6 02:53:20.952: default/client (ID:39536) -> default/server-jupiter (ID:21927) to-stack FORWARDED (ICMPv4 EchoRequest)
Nov  6 02:53:20.952: default/client (ID:39536) -> default/server-jupiter (ID:21927) to-endpoint FORWARDED (ICMPv4 EchoRequest)
Nov  6 02:53:20.952: default/client (ID:39536) <- default/server-jupiter (ID:21927) to-stack FORWARDED (ICMPv4 EchoReply)
Nov  6 02:53:20.952: default/client (ID:39536) <- default/server-jupiter (ID:21927) to-endpoint FORWARDED (ICMPv4 EchoReply)
Nov  6 02:59:37.925: default/client (ID:16946) -> default/server-default (ID:47491) to-stack FORWARDED (ICMPv4 EchoRequest)
Nov  6 02:59:37.925: default/client (ID:16946) <> default/server-default (ID:47491) policy-verdict:none INGRESS DENIED (ICMPv4 EchoRequest)
Nov  6 02:59:37.925: default/client (ID:16946) <> default/server-default (ID:47491) Policy denied DROPPED (ICMPv4 EchoRequest)
Nov  6 02:59:58.916: default/client (ID:39536) -> default/server-jupiter (ID:21927) to-stack FORWARDED (ICMPv4 EchoRequest)
Nov  6 02:59:58.916: default/client (ID:39536) <> default/server-jupiter (ID:21927) policy-verdict:none INGRESS DENIED (ICMPv4 EchoRequest)
Nov  6 02:59:58.916: default/client (ID:39536) <> default/server-jupiter (ID:21927) Policy denied DROPPED (ICMPv4 EchoRequest)
root@server:~# hubble observe  --pod client --protocol icmp --last 50 --verdict=DROPPED
Nov  6 02:59:37.925: default/client (ID:16946) <> default/server-default (ID:47491) policy-verdict:none INGRESS DENIED (ICMPv4 EchoRequest)
Nov  6 02:59:37.925: default/client (ID:16946) <> default/server-default (ID:47491) Policy denied DROPPED (ICMPv4 EchoRequest)
Nov  6 02:59:58.916: default/client (ID:39536) <> default/server-jupiter (ID:21927) policy-verdict:none INGRESS DENIED (ICMPv4 EchoRequest)
Nov  6 02:59:58.916: default/client (ID:39536) <> default/server-jupiter (ID:21927) Policy denied DROPPED (ICMPv4 EchoRequest)
root@server:~# CLIENT_ID=$(kubectl get cep client -o jsonpath='{.status.identity.id}')
CLIENT_CIL1_ID=$(kubectl get cep client-cil1 -o jsonpath='{.status.identity.id}')
echo $CLIENT_ID
echo $CLIENT_CIL1_ID
16946
39536
root@server:~# hubble observe  --pod client --protocol icmp --last 50 --verdict=DROPPED --identity $CLIENT_ID
Nov  6 02:59:37.925: default/client (ID:16946) <> default/server-default (ID:47491) policy-verdict:none INGRESS DENIED (ICMPv4 EchoRequest)
Nov  6 02:59:37.925: default/client (ID:16946) <> default/server-default (ID:47491) Policy denied DROPPED (ICMPv4 EchoRequest)
root@server:~# hubble observe --http-status 200
Nov  6 02:59:23.648: default/client:59092 (ID:16946) <- default/server-default:80 (ID:47491) http-response FORWARDED (HTTP/1.1 200 2ms (GET http://10.10.2.245/client-ip))
Nov  6 03:00:09.214: default/client-cil1:50406 (ID:39536) <- default/server-jupiter:80 (ID:21927) http-response FORWARDED (HTTP/1.1 200 6ms (GET http://192.168.16.20/public))
root@server:~# 

```

### 🥋 Exam Challenge
This exam is simple: you will need to create a multi-networked Pod called exam-client and connect it to both the existing default and jupiter networks.

Record the Identity ID for each endpoints for exam-client and exam-client-cil1 in the answers.yaml file.

You will pass the exam if the recorded endpoints Identity ID are correct.

Good luck!

ⓘ Notes:

You have access to the Editor.
All files used earlier in the lab are still available. Don't hesitate to check them!
You can also use the history button to check the commands you run previously.
Good luck!

🎓 You've done it!
On completion of this lab, you will receive a badge. Feel free to share your achievement on social media!

We hope that this lab gave you a first insight into how Cilium Multi Networking can help you with your more advanced Kubernetes networking requirements.

Don't forget to rate this lab and head over to our home page if you want to learn more! 

### Answer

```shell
EXAM_CLIENT_ID=$(kubectl get cep exam-client -o jsonpath='{.status.identity.id}')
EXAM_CLIENT_CIL1_ID=$(kubectl get cep exam-client-cil1 -o jsonpath='{.status.identity.id}')
echo $EXAM_CLIENT_ID
echo $EXAM_CLIENT_CIL1_ID
```