# Migrating from calico to cilium

https://isovalent.com/labs/cilium-migrating-from-calico/

## Migrating from Calico
Migrating to Cilium from another CNI is a very common task. But how do we minimize the impact during the migration? How do we ensure pods on the legacy CNI can still communicate to Cilium-managed during pods during the migration? How do we execute the migration safely, while avoiding a overly complex approach or using a separate tool such as Multus?

With the use of the new Cilium CRD CiliumNodeConfig, running clusters can be migrated on a node-by-node basis, without disrupting existing traffic or requiring a complete cluster outage or rebuild.

In this lab, you will migrate your cluster from Calico to Cilium.

Difficulty
Intermediate
Version
Open Source
Topics
Networking
Project
Cilium


## 🚚 Migrating to Cilium

Migrating to Cilium from another CNI is a very common task. But how do we minimize the impact during the migration? How do we ensure pods on the legacy CNI can still communicate to Cilium-managed pods during the migration? How do we execute the migration safely, while avoiding a overly complex approach or using a separe tool such as Multus?

With the use of the new Cilium CRD CiliumNodeConfig, running clusters can be migrated on a node-by-node basis, without disrupting existing traffic or requiring a complete cluster outage or rebuild.

In this lab, you will migrate your cluster from an existing CNI to Cilium. While we use Calico in this lab, you can leverage the same approach for other CNIs (albeit migrating from another CNI might require more efforts).

Note that this feature is still beta and we expect further tooling and automation to be developed to support large cluster migrations.

```bash
root@server:~# yq cluster.yaml
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      # nodepinger
      - containerPort: 32042
        hostPort: 32042
      # goldpinger
      - containerPort: 32043
        hostPort: 32043
  - role: worker
  - role: worker
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
root@server:~# ll
total 68
drwx------  6 root root  4096 Aug 10 12:47 ./
drwxr-xr-x 22 root root  4096 Aug 10 12:16 ../
-rw-r--r--  1 root root    16 Jul 23 14:52 .bash_aliases
-rw-r--r--  1 root root    16 Aug 10 12:51 .bash_history
-rw-r--r--  1 root root  1708 Aug 10 12:16 .bashrc
drwx------  3 root root  4096 Jul 23 14:52 .cache/
drwxr-xr-x  3 root root  4096 Jul 23 14:52 .config/
-rw-r--r--  1 root root   337 Jul 23 14:58 .gitconfig
drwxr-xr-x  3 root root  4096 Aug 10 12:48 .kube/
-rw-r--r--  1 root root   161 Apr 22 13:04 .profile
drwx------  2 root root  4096 Aug 10 12:16 .ssh/
-rw-------  1 root root 14583 Aug 10 12:16 .vimrc
-rw-r--r--  1 root root   215 Jul 23 14:58 .wget-hsts
-rw-r--r--  1 root root   349 Aug 10 12:47 cluster.yaml
root@server:~# kubectl get nodes
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   4m49s   v1.29.2
kind-worker          Ready    <none>          4m26s   v1.29.2
kind-worker2         Ready    <none>          4m26s   v1.29.2
kind-worker3         Ready    <none>          4m27s   v1.29.2
kind-worker4         Ready    <none>          4m30s   v1.29.2
root@server:~# kubectl -n calico-system rollout status ds/calico-node
daemon set "calico-node" successfully rolled out
root@server:~# kubectl -n calico-system get pod -o wide
NAME                                       READY   STATUS    RESTARTS   AGE     IP                NODE                 NOMINATED NODE   READINESS GATES
calico-kube-controllers-85955d4f5b-5j7hr   1/1     Running   0          5m4s    192.168.82.5      kind-control-plane   <none>           <none>
calico-node-c28hl                          1/1     Running   0          5m4s    172.18.0.5        kind-worker2         <none>           <none>
calico-node-cdljh                          1/1     Running   0          5m4s    172.18.0.2        kind-worker3         <none>           <none>
calico-node-dtv4k                          1/1     Running   0          5m4s    172.18.0.6        kind-worker4         <none>           <none>
calico-node-rhqwt                          1/1     Running   0          5m4s    172.18.0.4        kind-control-plane   <none>           <none>
calico-node-srjhn                          1/1     Running   0          5m4s    172.18.0.3        kind-worker          <none>           <none>
calico-typha-845fcccf46-226wr              1/1     Running   0          5m4s    172.18.0.5        kind-worker2         <none>           <none>
calico-typha-845fcccf46-594jl              1/1     Running   0          4m55s   172.18.0.2        kind-worker3         <none>           <none>
calico-typha-845fcccf46-qsl86              1/1     Running   0          4m55s   172.18.0.6        kind-worker4         <none>           <none>
csi-node-driver-cs2m6                      2/2     Running   0          5m4s    192.168.82.4      kind-control-plane   <none>           <none>
csi-node-driver-gbkxw                      2/2     Running   0          5m4s    192.168.195.194   kind-worker3         <none>           <none>
csi-node-driver-mzvs7                      2/2     Running   0          5m4s    192.168.110.130   kind-worker2         <none>           <none>
csi-node-driver-ntn6j                      2/2     Running   0          5m4s    192.168.218.194   kind-worker4         <none>           <none>
csi-node-driver-rhgvr                      2/2     Running   0          5m4s    192.168.162.130   kind-worker          <none>           <none>
root@server:~# kubectl get ipamblocks.crd.projectcalico.org \
  -o jsonpath="{range .items[*]}{'podNetwork: '}{.spec.cidr}{'\t NodeIP: '}{.spec.affinity}{'\n'}{end}"
podNetwork: 192.168.110.128/26   NodeIP: host:kind-worker2
podNetwork: 192.168.162.128/26   NodeIP: host:kind-worker
podNetwork: 192.168.195.192/26   NodeIP: host:kind-worker3
podNetwork: 192.168.218.192/26   NodeIP: host:kind-worker4
podNetwork: 192.168.82.0/26      NodeIP: host:kind-control-plane
root@server:~# kubectl rollout status daemonset nodepinger-goldpinger
kubectl get po -l app.kubernetes.io/instance=nodepinger -o wide
daemon set "nodepinger-goldpinger" successfully rolled out
NAME                          READY   STATUS    RESTARTS   AGE     IP                NODE                 NOMINATED NODE   READINESS GATES
nodepinger-goldpinger-84zng   1/1     Running   0          6m3s    192.168.82.2      kind-control-plane   <none>           <none>
nodepinger-goldpinger-f8n8q   1/1     Running   0          6m1s    192.168.218.193   kind-worker4         <none>           <none>
nodepinger-goldpinger-gx4cj   1/1     Running   0          6m1s    192.168.162.129   kind-worker          <none>           <none>
nodepinger-goldpinger-jg2f9   1/1     Running   0          5m55s   192.168.110.129   kind-worker2         <none>           <none>
nodepinger-goldpinger-sdrq2   1/1     Running   0          6m2s    192.168.195.193   kind-worker3         <none>           <none>
root@server:~# curl -s http://localhost:32042/check | jq
{
  "podResults": {
    "nodepinger-goldpinger-84zng": {
      "HostIP": "172.18.0.4",
      "OK": true,
      "PingTime": "2024-08-10T12:55:03.935Z",
      "PodIP": "192.168.82.2",
      "response": {
        "boot_time": "2024-08-10T12:48:47.470Z"
      },
      "status-code": 200
    },
    "nodepinger-goldpinger-f8n8q": {
      "HostIP": "172.18.0.6",
      "OK": true,
      "PingTime": "2024-08-10T12:55:08.526Z",
      "PodIP": "192.168.218.193",
      "response": {
        "boot_time": "2024-08-10T12:48:46.884Z"
      },
      "status-code": 200
    },
    "nodepinger-goldpinger-gx4cj": {
      "HostIP": "172.18.0.3",
      "OK": true,
      "PingTime": "2024-08-10T12:54:46.537Z",
      "PodIP": "192.168.162.129",
      "response": {
        "boot_time": "2024-08-10T12:48:47.270Z"
      },
      "status-code": 200
    },
    "nodepinger-goldpinger-jg2f9": {
      "HostIP": "172.18.0.5",
      "OK": true,
      "PingTime": "2024-08-10T12:55:02.202Z",
      "PodIP": "192.168.110.129",
      "response": {
        "boot_time": "2024-08-10T12:48:53.189Z"
      },
      "status-code": 200
    },
    "nodepinger-goldpinger-sdrq2": {
      "HostIP": "172.18.0.2",
      "OK": true,
      "PingTime": "2024-08-10T12:54:53.299Z",
      "PodIP": "192.168.195.193",
      "response": {
        "boot_time": "2024-08-10T12:48:49.014Z"
      },
      "status-code": 200
    }
  }
}
root@server:~# curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
goldpinger_nodes_health_total{goldpinger_instance="kind-worker2",status="healthy"} 5
goldpinger_nodes_health_total{goldpinger_instance="kind-worker2",status="unhealthy"} 0
root@server:~# kubectl apply -f /tmp/goldpinger_deploy.yaml
deployment.apps/goldpinger created
root@server:~# k get pod
NAME                          READY   STATUS    RESTARTS   AGE
goldpinger-57dfcb7b86-58m2t   1/1     Running   0          12s
goldpinger-57dfcb7b86-5nkk4   1/1     Running   0          12s
goldpinger-57dfcb7b86-8rmjq   1/1     Running   0          12s
goldpinger-57dfcb7b86-bf6z6   1/1     Running   0          12s
goldpinger-57dfcb7b86-crjjs   1/1     Running   0          12s
goldpinger-57dfcb7b86-ljcrl   1/1     Running   0          12s
goldpinger-57dfcb7b86-qlc7j   1/1     Running   0          12s
goldpinger-57dfcb7b86-s4nmq   1/1     Running   0          12s
goldpinger-57dfcb7b86-stljl   1/1     Running   0          12s
goldpinger-57dfcb7b86-zxd9x   1/1     Running   0          12s
nodepinger-goldpinger-84zng   1/1     Running   0          8m19s
nodepinger-goldpinger-f8n8q   1/1     Running   0          8m17s
nodepinger-goldpinger-gx4cj   1/1     Running   0          8m17s
nodepinger-goldpinger-jg2f9   1/1     Running   0          8m11s
nodepinger-goldpinger-sdrq2   1/1     Running   0          8m18s
root@server:~# k ctx
error: unknown command "ctx" for "kubectl"

Did you mean this?
        cp
root@server:~# k get pod -A
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
calico-apiserver     calico-apiserver-65f68d4985-46lh5            1/1     Running   0          8m6s
calico-apiserver     calico-apiserver-65f68d4985-56z2p            1/1     Running   0          8m6s
calico-system        calico-kube-controllers-85955d4f5b-5j7hr     1/1     Running   0          8m56s
calico-system        calico-node-c28hl                            1/1     Running   0          8m56s
calico-system        calico-node-cdljh                            1/1     Running   0          8m56s
calico-system        calico-node-dtv4k                            1/1     Running   0          8m56s
calico-system        calico-node-rhqwt                            1/1     Running   0          8m56s
calico-system        calico-node-srjhn                            1/1     Running   0          8m56s
calico-system        calico-typha-845fcccf46-226wr                1/1     Running   0          8m56s
calico-system        calico-typha-845fcccf46-594jl                1/1     Running   0          8m47s
calico-system        calico-typha-845fcccf46-qsl86                1/1     Running   0          8m47s
calico-system        csi-node-driver-cs2m6                        2/2     Running   0          8m56s
calico-system        csi-node-driver-gbkxw                        2/2     Running   0          8m56s
calico-system        csi-node-driver-mzvs7                        2/2     Running   0          8m56s
calico-system        csi-node-driver-ntn6j                        2/2     Running   0          8m56s
calico-system        csi-node-driver-rhgvr                        2/2     Running   0          8m56s
default              goldpinger-57dfcb7b86-58m2t                  1/1     Running   0          33s
default              goldpinger-57dfcb7b86-5nkk4                  1/1     Running   0          33s
default              goldpinger-57dfcb7b86-8rmjq                  1/1     Running   0          33s
default              goldpinger-57dfcb7b86-bf6z6                  1/1     Running   0          33s
default              goldpinger-57dfcb7b86-crjjs                  1/1     Running   0          33s
default              goldpinger-57dfcb7b86-ljcrl                  1/1     Running   0          33s
default              goldpinger-57dfcb7b86-qlc7j                  1/1     Running   0          33s
default              goldpinger-57dfcb7b86-s4nmq                  1/1     Running   0          33s
default              goldpinger-57dfcb7b86-stljl                  1/1     Running   0          33s
default              goldpinger-57dfcb7b86-zxd9x                  1/1     Running   0          33s
default              nodepinger-goldpinger-84zng                  1/1     Running   0          8m40s
default              nodepinger-goldpinger-f8n8q                  1/1     Running   0          8m38s
default              nodepinger-goldpinger-gx4cj                  1/1     Running   0          8m38s
default              nodepinger-goldpinger-jg2f9                  1/1     Running   0          8m32s
default              nodepinger-goldpinger-sdrq2                  1/1     Running   0          8m39s
kube-system          coredns-76f75df574-g9nq4                     1/1     Running   0          9m10s
kube-system          coredns-76f75df574-x7tl4                     1/1     Running   0          9m10s
kube-system          etcd-kind-control-plane                      1/1     Running   0          9m24s
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          9m24s
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          9m24s
kube-system          kube-proxy-jvcng                             1/1     Running   0          9m8s
kube-system          kube-proxy-mzs2j                             1/1     Running   0          9m10s
kube-system          kube-proxy-pmhnr                             1/1     Running   0          9m4s
kube-system          kube-proxy-r7fvz                             1/1     Running   0          9m4s
kube-system          kube-proxy-wvmtc                             1/1     Running   0          9m5s
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          9m24s
local-path-storage   local-path-provisioner-7577fdbbfb-s8tbf      1/1     Running   0          9m10s
tigera-operator      tigera-operator-774b5dd976-nnxgb             1/1     Running   0          9m2s
root@server:~# kubectl rollout status deployment goldpinger
kubectl get po -l app=goldpinger -o wide
deployment "goldpinger" successfully rolled out
NAME                          READY   STATUS    RESTARTS   AGE   IP                NODE                 NOMINATED NODE   READINESS GATES
goldpinger-57dfcb7b86-58m2t   1/1     Running   0          73s   192.168.110.131   kind-worker2         <none>           <none>
goldpinger-57dfcb7b86-5nkk4   1/1     Running   0          73s   192.168.195.195   kind-worker3         <none>           <none>
goldpinger-57dfcb7b86-8rmjq   1/1     Running   0          73s   192.168.162.132   kind-worker          <none>           <none>
goldpinger-57dfcb7b86-bf6z6   1/1     Running   0          73s   192.168.82.7      kind-control-plane   <none>           <none>
goldpinger-57dfcb7b86-crjjs   1/1     Running   0          73s   192.168.110.132   kind-worker2         <none>           <none>
goldpinger-57dfcb7b86-ljcrl   1/1     Running   0          73s   192.168.162.133   kind-worker          <none>           <none>
goldpinger-57dfcb7b86-qlc7j   1/1     Running   0          73s   192.168.82.8      kind-control-plane   <none>           <none>
goldpinger-57dfcb7b86-s4nmq   1/1     Running   0          73s   192.168.218.197   kind-worker4         <none>           <none>
goldpinger-57dfcb7b86-stljl   1/1     Running   0          73s   192.168.218.196   kind-worker4         <none>           <none>
goldpinger-57dfcb7b86-zxd9x   1/1     Running   0          73s   192.168.195.196   kind-worker3         <none>           <none>
root@server:~# kubectl expose deployment goldpinger --type NodePort \
  --overrides '{"spec":{"ports": [{"port":80,"protocol":"TCP","targetPort":8080,"nodePort":32043}]}}'
service/goldpinger exposed
root@server:~# curl -s http://localhost:32043/check | jq
{
  "podResults": {
    "goldpinger-57dfcb7b86-58m2t": {
      "HostIP": "172.18.0.5",
      "OK": true,
      "PingTime": "2024-08-10T12:58:50.525Z",
      "PodIP": "192.168.110.131",
      "response": {
        "boot_time": "2024-08-10T12:56:39.921Z"
      },
      "status-code": 200
    },
    "goldpinger-57dfcb7b86-5nkk4": {
      "HostIP": "172.18.0.2",
      "OK": true,
      "PingTime": "2024-08-10T12:58:55.512Z",
      "PodIP": "192.168.195.195",
      "response": {
        "boot_time": "2024-08-10T12:56:39.922Z"
      },
      "status-code": 200
    },
    "goldpinger-57dfcb7b86-8rmjq": {
      "HostIP": "172.18.0.3",
      "OK": true,
      "PingTime": "2024-08-10T12:59:00.214Z",
      "PodIP": "192.168.162.132",
      "response": {
        "boot_time": "2024-08-10T12:56:39.924Z"
      },
      "status-code": 200
    },
    "goldpinger-57dfcb7b86-bf6z6": {
      "HostIP": "172.18.0.4",
      "OK": true,
      "PingTime": "2024-08-10T12:59:07.117Z",
      "PodIP": "192.168.82.7",
      "response": {
        "boot_time": "2024-08-10T12:56:39.922Z"
      },
      "status-code": 200
    },
    "goldpinger-57dfcb7b86-crjjs": {
      "HostIP": "172.18.0.5",
      "OK": true,
      "PingTime": "2024-08-10T12:59:07.984Z",
      "PodIP": "192.168.110.132",
      "response": {
        "boot_time": "2024-08-10T12:56:40.576Z"
      },
      "status-code": 200
    },
    "goldpinger-57dfcb7b86-ljcrl": {
      "HostIP": "172.18.0.3",
      "OK": true,
      "PingTime": "2024-08-10T12:58:42.557Z",
      "PodIP": "192.168.162.133",
      "response": {
        "boot_time": "2024-08-10T12:56:40.612Z"
      },
      "status-code": 200
    },
    "goldpinger-57dfcb7b86-qlc7j": {
      "HostIP": "172.18.0.4",
      "OK": true,
      "PingTime": "2024-08-10T12:59:00.971Z",
      "PodIP": "192.168.82.8",
      "response": {
        "boot_time": "2024-08-10T12:56:40.629Z"
      },
      "status-code": 200
    },
    "goldpinger-57dfcb7b86-s4nmq": {
      "HostIP": "172.18.0.6",
      "OK": true,
      "PingTime": "2024-08-10T12:58:47.641Z",
      "PodIP": "192.168.218.197",
      "response": {
        "boot_time": "2024-08-10T12:56:40.521Z"
      },
      "response-time-ms": 1,
      "status-code": 200
    },
    "goldpinger-57dfcb7b86-stljl": {
      "HostIP": "172.18.0.6",
      "OK": true,
      "PingTime": "2024-08-10T12:58:43.490Z",
      "PodIP": "192.168.218.196",
      "response": {
        "boot_time": "2024-08-10T12:56:39.876Z"
      },
      "status-code": 200
    },
    "goldpinger-57dfcb7b86-zxd9x": {
      "HostIP": "172.18.0.2",
      "OK": true,
      "PingTime": "2024-08-10T12:58:54.481Z",
      "PodIP": "192.168.195.196",
      "response": {
        "boot_time": "2024-08-10T12:56:40.599Z"
      },
      "status-code": 200
    }
  }
}
root@server:~# curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
goldpinger_nodes_health_total{goldpinger_instance="kind-worker2",status="healthy"} 10
goldpinger_nodes_health_total{goldpinger_instance="kind-worker2",status="unhealthy"} 0
root@server:~# 
```

## Networking

In the networking section of the configuration file, the default CNI had been disabled. Instead, Calico was deployed to the cluster to provide network connectivity.

To check that the Kind cluster was successfully installed, verify that the nodes are up and joined:

shell

copy

run
kubectl get nodes
You should see 1 control-plane and 4 nodes appear, all marked as Ready.

Check that Calico was successfully deployed by looking at the status of the Calico DaemonSet:

shell

copy

run
kubectl -n calico-system rollout status ds/calico-node
Let's check the PodCIDR on each node. (Note: the PodCIDR is the range the Pods will get an IP address from).

shell

copy

run
kubectl get ipamblocks.crd.projectcalico.org \
  -o jsonpath="{range .items[*]}{'podNetwork: '}{.spec.cidr}{'\t NodeIP: '}{.spec.affinity}{'\n'}{end}"

## Goldpinger

We have deployed a software called Goldpinger on the cluster. Goldpinger deploys one pod per node (with a DaemonSet) to monitor node connectivity.

Check that the Nodepinger Daemonset is running properly:

shell

copy

run
kubectl rollout status daemonset nodepinger-goldpinger
kubectl get po -l app.kubernetes.io/instance=nodepinger -o wide
Go to the 🔗 Nodepinger tab to see the graph generated by these pods.

If necessary, reload the tab by clicking on the reload icon:

Nodepinger

We can also check the status in the >_ Terminal tab with curl:

shell

copy

run
curl -s http://localhost:32042/check | jq
The status code should be 200 for all 5 nodes.

Finally, let's check metrics as it's easier to see how many nodes are healthy or unhealthy:

shell

copy

run
curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
You should see 5 healthy nodes (if it doesn't, try again).


## Pods on every nodes

The first Goldpinger deployment (which we called "Nodepinger") is practical, but since it uses DaemonSets, these pods won't get deleted when we drain nodes for the migration.

To show the difference, we'll also deploy Goldpinger a second time as a Deployment with 10 replicas:

shell

copy

run
kubectl apply -f /tmp/goldpinger_deploy.yaml
Check that the pods are running and spread on all 5 nodes:

shell

copy

run
kubectl rollout status deployment goldpinger
kubectl get po -l app=goldpinger -o wide
Then expose the pods as a service on NodePort 32043:

shell

copy

run
kubectl expose deployment goldpinger --type NodePort \
  --overrides '{"spec":{"ports": [{"port":80,"protocol":"TCP","targetPort":8080,"nodePort":32043}]}}'
Check the status of these pods by looking at the 🔗 Goldpinger tab.

Goldpinger

Refresh the tab if necessary by clicking on the reload icon in the top right corner.

Back in the >_ Terminal, you can also check the status with curl:

shell

copy

run
curl -s http://localhost:32043/check | jq
Or using the metrics:

shell

copy

run
curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
You should see 10 healthy and 0 unhealthy pods (if it doesn't, try again).

In the next tasks, we will prepare the cluster for the migration from Calico to Cilium.

## 🧩 Kubelet and CNI plugins
On every node, the Kubelet is configured (via files in the /etc/cni/net.d/ directory) to use one (or several) CNI plugins.

The CNI plugins handle networking tasks for Pods:

allocating IP address(es)
creating & configuring the Pod's network interface(s)
(potentially) establishing an overlay network
The Pod’s network configuration shares the same life cycle as the Pod Sandbox.

When migrating CNI plugins, several approaches are possible, with pros and cons.

## 🛤️ Migration Approaches
The ideal scenario is to build a brand new cluster and migrate workloads using a GitOps approach. This can however involve a lot of preparation work and potential disruptions.
Another method consists in reconfiguring /etc/cni/net.d/ to point to Cilium. However, existing Pods will still have been configured by the old network plugin while new Pods will be configured by the newer CNI plugin. To complete the migration, all Pods on the cluster that are configured by the old CNI must be recycled in order to be managed by the new CNI plugin.
A naive approach to migrating a CNI would be to reconfigure all nodes with a new CNI and then gradually restart each node in the cluster, thus replacing the CNI when the node is brought back up and ensuring that all pods are part of the new CNI. This simple migration, while effective, comes at the cost of disrupting cluster connectivity during the rollout. Unmigrated and migrated nodes would be split in to two “islands” of connectivity, and pods would be randomly unable to reach one-another until the migration is complete.
In this lab, you will learn about a new approach.

## 🎭 Migration via dual overlays
Cilium supports a hybrid mode, where two separate overlays are established across the cluster. While Pods on a given node can only be attached to one network, they have access to both Cilium and non-Cilium pods while the migration is taking place. As long as Cilium and the existing networking provider use a separate IP range, the Linux routing table takes care of separating traffic.

In this lab, we will use a model for live migrating between two deployed CNI implementations. This will have the benefit of reducing downtime of nodes and workloads and ensuring that workloads on both configured CNIs can communicate during migration.

For live migration to work, Cilium will be installed with a separate CIDR range and encapsulation port than that of the currently installed CNI. As long as Cilium and the existing CNI use a separate IP range, the Linux routing table takes care of separating traffic.

## 📋 Requirements
Live migration requires the following:

An existing CNI plugin that uses the Linux routing stack, such as Flannel, Calico, or AWS-CNI
A Cilium Pod CIDR distinct from the previous CNI plugin
A Cilium overlay distinct from the previous CNI plugin, either by changing the protocol or port
Use of the Cilium Cluster Pool IPAM mode

## 🗺️ Migration Overview
The migration process utilizes the per-node configuration feature to selectively enable Cilium CNI. This allows for a controlled rollout of Cilium without disrupting existing workloads.

Cilium will be installed, first, in a mode where it establishes an overlay but does not provide CNI networking for any pods. Then, individual nodes will be migrated.

In summary, the process looks like:

Prepare the cluster and install Cilium in “secondary” mode.
Cordon, drain, migrate, and reboot each node
Remove the existing network provider
(Optional) Reboot each node again
In the next task, you will prepare the cluster and install Cilium in "secondary" mode.

### Preparation: Pod CIDR

The first step is to select a new CIDR for pods. It must be distinct from all other CIDRs in use.

Let's check the CIDR used by Calico:

shell

copy

run
kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.ipPools[*].cidr}{"\n"}'
Calico is using its default CIDR, which is 192.168.0.0/16. In order to avoid conflicts, we will use 10.244.0.0/16 —which is the usual default on Kind— as the pod CIDR for Cilium.




### Preparation: Encapsulation Protocol

The second step is to select a different encapsulation protocol (Geneve instead of VXLAN for example) or a distinct encapsulation port.

Check which encapsulation protocol Calico is using:

shell

copy

run
kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.ipPools[*].encapsulation}{"\n"}'
Calico is using VXLANCrossSubnet. In order to avoid clashing with Calico's VXLAN port, we need to use VXLAN with a non-default port. Since the standard port is 8472, let's use 8473 instead.

### Cilium Values for Migration

We have pre-created a Cilium Helm configuration file values-migration.yaml based on the details above:

Review it:

shell

copy

run
yq values-migration.yaml
Let's review some of the key parameters first:

🔧 Operator Configuration
yaml
operator:
  unmanagedPodWatcher:
    restart: false
This is there to prevent the Cilium Operator from restarting Pods that are not being managed by Cilium (we don't want to disrupt the pods that are not managed by Cilium).

🔌 VXLAN Configuration
yaml
tunnelPort: 8473
As highlighted earlier, this is to prevent Cilium's VXLAN from conflicting with Calico's.

🌐 CNI Configuration
yaml
cni:
  customConf: true
  uninstall: false
The first setting (customConf: true) above temporarily skips writing the CNI configuration. This is to prevent Cilium from taking over immediately.

Once the migration is complete, we will switch customConf back to the default false value.

The second setting (uninstall: false) above will prevent Cilium from removing the CNI configuration file and plugin binaries for Calico, thus allowing the temporary migration state.

🪪 IPAM Configuration
yaml
ipam:
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDRList: ["10.244.0.0/16"]
Live migration requires to use the Cluster Pool IPAM mode, and a Pod CIDR distinct from Calico's in order to perform the migration.

🛡️ Network Policies Configuration
yaml
policyEnforcementMode: "never"
The above disables the enforcement of network policy until the migration is completed. We will enforce network policies post-migration.

🐝 BPF Configuration
yaml
bpf:
  hostLegacyRouting: true
This flag routes traffic via the host stack to provide connectivity during the migration. We will verify during the migration that Calico-managed pods and Cilium-managed pods have connectivity.

### Generate Cilium Hel Values

In this lab, we will use cilium-cli to auto-detect settings specific to the underlying cluster platform (Kind in this case) and generate Helm values. We will then use the Helm chart to install Cilium.

Let's generate the values-migration.yaml Helm values file:

shell

copy

run
cilium install \
  --helm-values values-migration.yaml \
  --dry-run-helm-values > values-initial.yaml
The above command:

uses Helm values from values-migration.yaml
automatically fills in the missing values through the use of the helm-auto-gen-values flag
creates a new Helm values file called values-initial.yaml
Review the created file:

shell

copy

run
yq values-initial.yaml
It is a combination of the values pulled from the values-migration.yaml file and the one auto-generated by the Cilium CLI, such as:

operator.replicas
cluster.id and cluster.name
kubeProxyReplacement
the serviceAccounts section
encryption.nodeEncryption

### Prevent CAlico from using Cilium Interfaces

Calico has several modes of operation to decide which network interface to use by default on nodes that it manages. This is configured using the Tigera Operator's spec.calicoNetwork.nodeAddressAutodetectionV4 (and respectively nodeAddressAutodetectionV6 for IPv6) parameter.

By default, it is set to firstFound: true, which will use the first detected network interface on the node. Check this value with:

shell

copy

run
kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.nodeAddressAutodetectionV4}{"\n"}'
When installing Cilium on the nodes, Cilium will create a new network interface called cilium_host. If Calico decides to use it as its default interface, Calico node routing will start failing. For this reason, we want to make sure that Calico ignores the cilium_host interface.

Depending on your Tigera Operator settings (for example if you use the interface or skipInterface options), you might want to adjust the parameters to ensure cilium_host is not considered.

In our lab, we will simply set firstFound to false and use kubernetes: NodeInternalIP instead, so Calico uses the node's internal IP as its main interface. Patch the Tigera Operator's configuration with:

shell

copy

run
kubectl patch installations.operator.tigera.io default --type=merge \
  --patch '{"spec": {"calicoNetwork": {"nodeAddressAutodetectionV4": {"firstFound": false, "kubernetes": "NodeInternalIP"}}}}'
And check the value again:

shell

copy

run
kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.nodeAddressAutodetectionV4}{"\n"}'

### Install Cilium

Let's now install Cilium using helm and the values we have just generated.

shell

copy

run
helm repo add cilium https://helm.cilium.io/
helm upgrade --install --namespace kube-system cilium cilium/cilium \
  --values values-initial.yaml
Verify that Cilium is properly installed (this might take a few minutes):

shell

copy

run
cilium status --wait
Note that the Cluster Pods entry indicates that no pods are managed by Cilium. While Cilium is installed on every node and an overlay is established between the nodes, it is not yet configured to manage pods on nodes.

Check the CNI configuration on one of the nodes:

shell

copy

run
docker exec kind-worker ls /etc/cni/net.d/
It still only contains Calico configuration, so Kubelets are unable to make use of Cilium for now.

### CiliumNodeConfig

In a standard installation, all Cilium agents have the same configuration, controlled by the cilium-config ConfigMap resource.

However in our migration, we want to rollout Cilium on one node at a time. In order to achieve this, we will use the CiliumNodeConfig resource type (a CRD that was added in Cilium 1.13), which allows to configure Cilium agents on a per-node basis.

A CiliumNodeConfig object consists of a set of fields and a label selector. The label selector defines to which nodes the configuration applies.

Let's now create a per-node config that will instruct Cilium to “take over” CNI networking on the node.

Using the </> Editor tab, paste the following code into the ciliumnodeconfig.yaml file:

yaml

copy
apiVersion: cilium.io/v2alpha1
kind: CiliumNodeConfig
metadata:
  namespace: kube-system
  name: cilium-default
spec:
  nodeSelector:
    matchLabels:
      io.cilium.migration/cilium-default: "true"
  defaults:
    write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
    custom-cni-conf: "false"
    cni-chaining-mode: "none"
    cni-exclusive: "true"
This configuration will be used to switch the nodes' CNI configuration from Calico to Cilium.

When the file is saved, head back to the >_ Terminal tab and apply it:

shell

copy

run
kubectl apply --server-side -f ciliumnodeconfig.yaml
Check the list of nodes on the cluster, with their labels:

shell

copy

run
kubectl get no --show-labels
There is currently no nodes matching the io.cilium.migration/cilium-default: "true" condition in our CiliumNodeConfig resource. For this reason, this configuration does not apply to any nodes for now.

In the next challenge, we will start migrating nodes by applying this label to them, one by one.

### Console output

```bash
root@server:~# kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.ipPools[*].cidr}{"\n"}'
192.168.0.0/16
root@server:~# kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.ipPools[*].encapsulation}{"\n"}'
VXLANCrossSubnet
root@server:~# yq values-migration.yaml
---
operator:
  unmanagedPodWatcher:
    restart: false
tunnel: vxlan
tunnelPort: 8473
cni:
  customConf: true
  uninstall: false
ipam:
  mode: "cluster-pool"
  operator:
    clusterPoolIPv4PodCIDRList: ["10.244.0.0/16"]
policyEnforcementMode: "never"
bpf:
  hostLegacyRouting: true
root@server:~# cilium install \
  --helm-values values-migration.yaml \
  --dry-run-helm-values > values-initial.yaml
root@server:~# ll
total 76
drwx------  6 root root  4096 Aug 10 13:18 ./
drwxr-xr-x 22 root root  4096 Aug 10 12:16 ../
-rw-r--r--  1 root root    16 Jul 23 14:52 .bash_aliases
-rw-r--r--  1 root root  1317 Aug 10 13:18 .bash_history
-rw-r--r--  1 root root  1708 Aug 10 12:16 .bashrc
drwx------  3 root root  4096 Jul 23 14:52 .cache/
drwxr-xr-x  3 root root  4096 Jul 23 14:52 .config/
-rw-r--r--  1 root root   337 Jul 23 14:58 .gitconfig
drwxr-xr-x  3 root root  4096 Aug 10 12:48 .kube/
-rw-r--r--  1 root root   161 Apr 22 13:04 .profile
drwx------  2 root root  4096 Aug 10 12:16 .ssh/
-rw-------  1 root root 14583 Aug 10 12:16 .vimrc
-rw-r--r--  1 root root   215 Jul 23 14:58 .wget-hsts
-rw-r--r--  1 root root     0 Aug 10 13:02 ciliumnodeconfig.yaml
-rw-r--r--  1 root root   349 Aug 10 12:47 cluster.yaml
-rw-r--r--  1 root root   361 Aug 10 13:18 values-initial.yaml
-rw-r--r--  1 root root   283 Aug 10 13:02 values-migration.yaml
root@server:~# cat values-initial.yaml 
bpf:
  hostLegacyRouting: true
cluster:
  name: kind-kind
cni:
  customConf: true
  uninstall: false
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
    - 10.244.0.0/16
operator:
  replicas: 1
  unmanagedPodWatcher:
    restart: false
policyEnforcementMode: never
routingMode: tunnel
tunnel: vxlan
tunnelPort: 8473
tunnelProtocol: vxlan

root@server:~# yq values-initial.yaml
bpf:
  hostLegacyRouting: true
cluster:
  name: kind-kind
cni:
  customConf: true
  uninstall: false
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - 10.244.0.0/16
operator:
  replicas: 1
  unmanagedPodWatcher:
    restart: false
policyEnforcementMode: never
routingMode: tunnel
tunnel: vxlan
tunnelPort: 8473
tunnelProtocol: vxlan
root@server:~# kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.nodeAddressAutodetectionV4}{"\n"}'
{"firstFound":true}
root@server:~# kubectl patch installations.operator.tigera.io default --type=merge \
  --patch '{"spec": {"calicoNetwork": {"nodeAddressAutodetectionV4": {"firstFound": false, "kubernetes": "NodeInternalIP"}}}}'
installation.operator.tigera.io/default patched
root@server:~# kubectl get installations.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.nodeAddressAutodetectionV4}{"\n"}'
{"firstFound":false,"kubernetes":"NodeInternalIP"}
root@server:~# helm repo add cilium https://helm.cilium.io/
helm upgrade --install --namespace kube-system cilium cilium/cilium \
  --values values-initial.yaml
"cilium" has been added to your repositories
Release "cilium" does not exist. Installing it now.
NAME: cilium
LAST DEPLOYED: Sat Aug 10 13:24:02 2024
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You have successfully installed Cilium with Hubble.

Your release version is 1.16.0.

For any further help, visit https://docs.cilium.io/en/v1.16/gettinghelp
root@server:~# cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium-envoy       Desired: 5, Ready: 5/5, Available: 5/5
DaemonSet              cilium             Desired: 5, Ready: 5/5, Available: 5/5
Containers:            cilium-operator    Running: 1
                       cilium-envoy       Running: 5
                       cilium             Running: 5
Cluster Pods:          0/26 managed by Cilium
Helm chart version:    
Image versions         cilium             quay.io/cilium/cilium:v1.16.0@sha256:46ffa4ef3cf6d8885dcc4af5963b0683f7d59daa90d49ed9fb68d3b1627fe058: 5
                       cilium-operator    quay.io/cilium/operator-generic:v1.16.0@sha256:d6621c11c4e4943bf2998af7febe05be5ed6fdcf812b27ad4388f47022190316: 1
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.29.7-39a2a56bbd5b3a591f69dbca51d3e30ef97e0e51@sha256:bd5ff8c66716080028f414ec1cb4f7dc66f40d2fb5a009fff187f4a9b90b566b: 5
root@server:~# docker exec kind-worker ls /etc/cni/net.d/
10-calico.conflist
calico-kubeconfig
root@server:~# kubectl apply --server-side -f ciliumnodeconfig.yaml
Warning: cilium.io/v2alpha1 CiliumNodeConfig will be deprecated in cilium v1.16; use cilium.io/v2 CiliumNodeConfig
ciliumnodeconfig.cilium.io/cilium-default serverside-applied
root@server:~# kubectl get no --show-labels
NAME                 STATUS   ROLES           AGE   VERSION   LABELS
kind-control-plane   Ready    control-plane   45m   v1.29.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
kind-worker          Ready    <none>          44m   v1.29.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker,kubernetes.io/os=linux
kind-worker2         Ready    <none>          44m   v1.29.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker2,kubernetes.io/os=linux
kind-worker3         Ready    <none>          44m   v1.29.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker3,kubernetes.io/os=linux
kind-worker4         Ready    <none>          44m   v1.29.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker4,kubernetes.io/os=linux
root@server:~# 




```

## Progress
🚚 Migration
We are now ready to begin the migration process. We will do it a node at a time.

We will cordon, drain, migrate, and reboot each node.


### Introduction

Check the 🔗 Goldpinger tab to verify that the pods are still able to communicate with each other fine.




### Cordon and Drain the Node

It is recommended to always cordon and drain at the beginning of the migration process, so that end-users are not impacted by any potential issues.

Cordoning a node will prevent new pods from being scheduled on it.

Draining a node will gracefully evict all the running pods from the node. This ensures that the pods are not abruptly terminated and that their workload is gracefully handled by other available nodes.

Navigate back to the >_ Terminal tab.

Let's get started with the kind-worker node:

shell

copy

run
NODE="kind-worker"
Let's cordon the node:

shell

copy

run
kubectl cordon $NODE
Expect to see an output such as node/kind-worker cordoned.

Let's now drain the node. Note that we use the ignore-daemonset flag as several DaemonSets are still required to run. When we drain a node, the node is automatically cordoned. We cordoned first in this instance to provide clarity in the migration process, but you don't need to do both steps in the future.

shell

copy

run
kubectl drain $NODE --ignore-daemonsets
Expect to see an output such as:

node/kind-worker already cordoned
Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-h2x5m, calico-system/csi-node-driver-9j7vr, default/nodepinger-goldpinger-rzpvp, kube-system/cilium-c8ts4, kube-system/kube-proxy-6w6q9
evicting pod local-path-storage/local-path-provisioner-75f5b54ffd-6jcn5
evicting pod default/goldpinger-59b4d5ccd6-kqlp5
evicting pod calico-system/calico-typha-68b499c88f-ffwv2
evicting pod default/goldpinger-59b4d5ccd6-m62gs
evicting pod kube-system/coredns-787d4945fb-pb5q7
evicting pod default/goldpinger-59b4d5ccd6-9q465
pod/goldpinger-59b4d5ccd6-9q465 evicted
pod/goldpinger-59b4d5ccd6-m62gs evicted
pod/goldpinger-59b4d5ccd6-kqlp5 evicted
pod/coredns-787d4945fb-pb5q7 evicted
pod/calico-typha-68b499c88f-ffwv2 evicted
pod/local-path-provisioner-75f5b54ffd-6jcn5 evicted
node/kind-worker drained
Let's verify no pods are running on the drained node (besides the goldpinger pod which is part of a DaemonSet):

shell

copy

run
kubectl get pods -o wide --field-selector spec.nodeName=$NODE
Verify that the Nodepinger still sees 5 pods:

shell

copy

run
curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'

### Label and Restart

We can now label the node, which will cause the CiliumNodeConfig to apply to this node.

shell

copy

run
kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
Restart Cilium on the node. That will trigger the creation of CNI configuration file.

shell

copy

run
kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
Expect to see output such as:

pod "cilium-9gck7" deleted
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
Check the CNI configurations on the node:

shell

copy

run
docker exec $NODE ls /etc/cni/net.d/
Cilium has renamed the Calico configuration (10-calico.conflist) and deployed its own configuration file 05-cilium.conflist, so the Kubelet on that node is now configured to use Cilium as its CNI provider.

Check the Nodepinger pods:

shell

copy

run
kubectl get po -l app.kubernetes.io/instance=nodepinger \
  --field-selector spec.nodeName=$NODE -o wide
It is still using the Calico pod CIDR, since the pod was not restarted yet. Delete it to force it to restart:

shell

copy

run
kubectl delete po -l app.kubernetes.io/instance=nodepinger \
  --field-selector spec.nodeName=$NODE
Check it again:

shell

copy

run
kubectl get po -l app.kubernetes.io/instance=nodepinger \
  --field-selector spec.nodeName=$NODE -o wide
It has now received an IP from the Cilium pod CIDR range (10.244.0.0/16).

Verify that the connectivity is still fine:

shell

copy

run
curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'

### Verification

Let's check the Cilium status. It may take a minute or so.

shell

copy

run
cilium status --wait
The Cluster Pods section indicates that 1 Pod is managed by Cilium, the goldpinger Pod (since it is part of a DaemonSet).

Check the Pod CIDR that was assigned to the node by Cilium:

shell

copy

run

```bash
kubectl get ciliumnode kind-worker \
  -o jsonpath='{.spec.ipam.podCIDRs[0]}{"\n"}'
```

We can finally uncordon the node with:

shell

copy

run

```bash
kubectl uncordon $NODE
```

Scale the goldpinger deployment so it spreads on the newly migrated node:

shell

copy

run

```bash
kubectl scale deployment goldpinger --replicas 15
```

Check the assigned IPs for the new pods:

shell

copy

run

```bash
kubectl get po -l app=goldpinger --field-selector spec.nodeName=$NODE -o wide
```

Then check that all pings work fine with the previously deployed pods in the deployment (it might take a little while, relaunch the command):

shell

copy

run

```bash
curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
```

You can also check the 🔗 Goldpinger tab and refresh it to check that all 15 pods can talk to each other.

### Repeat for other worker nodes

Let's migrate all the other worker nodes, by looping on all of them:

shell

copy

run

```bash
for i in $(seq 2 4); do
  node="kind-worker${i}"
  echo "Migrating node ${node}"
  kubectl drain $node --ignore-daemonsets
  kubectl label node $node --overwrite "io.cilium.migration/cilium-default=true"
  kubectl -n kube-system delete pod --field-selector spec.nodeName=$node -l k8s-app=cilium
  kubectl -n kube-system rollout status ds/cilium -w
  kubectl uncordon $node
done
```

Check that the Cilium status is correct:

shell

copy

run

```bash
cilium status --wait
```

The Nodepinger DaemonSet pods will still be on the old Pod CIDR, so let's restart all of them:

shell

copy

run

```bash
kubectl rollout restart daemonset nodepinger-goldpinger
kubectl rollout status daemonset nodepinger-goldpinger
```

Now check that all pods on worker nodes have a Cilium IP address:

shell

copy

run

```bash
kubectl get po -o wide
```

Check that connectivity is still fine for both the DaemonSet and the Deployment Goldpinger apps:

shell

copy

run

```bash
curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
```

The only node left to migrate is the control plane.

### Repeat for kind-control-plane

We can now proceed to the migration of the final node.

Let's cordon the node:

shell

copy

run

```bash
NODE="kind-control-plane"
kubectl drain $NODE --ignore-daemonsets
kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
kubectl uncordon $NODE
```

Ensure the Nodepinger pods are all restarted:

shell

copy

run

```bash
kubectl rollout restart daemonset nodepinger-goldpinger
kubectl rollout status daemonset nodepinger-goldpinger
```

In addition, restart all csi-node-driver DaemonSet pods inside the calico-system namespace to complete the migration of all existing pods to Cilium:

shell

copy

run

```bash
kubectl rollout restart daemonset -n calico-system csi-node-driver
kubectl rollout status daemonset -n calico-system csi-node-driver
```
The status of Cilium should be OK and all pods should be managed by Cilium:

shell

copy

run

```bash
cilium status --wait
```

Now the migration of the nodes is complete, let's clean things up!

#### Console Output

```bash
root@server:~# NODE="kind-worker"
root@server:~# kubectl cordon $NODE
node/kind-worker cordoned
root@server:~# k get node
NAME                 STATUS                     ROLES           AGE   VERSION
kind-control-plane   Ready                      control-plane   49m   v1.29.2
kind-worker          Ready,SchedulingDisabled   <none>          49m   v1.29.2
kind-worker2         Ready                      <none>          49m   v1.29.2
kind-worker3         Ready                      <none>          49m   v1.29.2
kind-worker4         Ready                      <none>          49m   v1.29.2
root@server:~# kubectl drain $NODE --ignore-daemonsets
node/kind-worker already cordoned
Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-q9pmp, calico-system/csi-node-driver-rhgvr, default/nodepinger-goldpinger-gx4cj, kube-system/cilium-5vprv, kube-system/cilium-envoy-lxcrm, kube-system/kube-proxy-pmhnr
evicting pod default/goldpinger-57dfcb7b86-8rmjq
evicting pod default/goldpinger-57dfcb7b86-ljcrl
evicting pod calico-apiserver/calico-apiserver-65f68d4985-46lh5
evicting pod tigera-operator/tigera-operator-774b5dd976-nnxgb
pod/tigera-operator-774b5dd976-nnxgb evicted
pod/goldpinger-57dfcb7b86-ljcrl evicted
pod/goldpinger-57dfcb7b86-8rmjq evicted
pod/calico-apiserver-65f68d4985-46lh5 evicted
node/kind-worker drained
root@server:~# k get node
NAME                 STATUS                     ROLES           AGE   VERSION
kind-control-plane   Ready                      control-plane   50m   v1.29.2
kind-worker          Ready,SchedulingDisabled   <none>          49m   v1.29.2
kind-worker2         Ready                      <none>          49m   v1.29.2
kind-worker3         Ready                      <none>          49m   v1.29.2
kind-worker4         Ready                      <none>          49m   v1.29.2
root@server:~# kubectl get pods -o wide --field-selector spec.nodeName=$NODE
NAME                          READY   STATUS    RESTARTS   AGE   IP                NODE          NOMINATED NODE   READINESS GATES
nodepinger-goldpinger-gx4cj   1/1     Running   0          49m   192.168.162.129   kind-worker   <none>           <none>
root@server:~# curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
goldpinger_nodes_health_total{goldpinger_instance="kind-worker2",status="healthy"} 5
goldpinger_nodes_health_total{goldpinger_instance="kind-worker2",status="unhealthy"} 0
root@server:~# kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
node/kind-worker labeled
root@server:~# kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
pod "cilium-5vprv" deleted
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
root@server:~# docker exec $NODE ls /etc/cni/net.d/
05-cilium.conflist
10-calico.conflist.cilium_bak
calico-kubeconfig
root@server:~# kubectl get po -l app.kubernetes.io/instance=nodepinger \
  --field-selector spec.nodeName=$NODE -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP                NODE          NOMINATED NODE   READINESS GATES
nodepinger-goldpinger-gx4cj   1/1     Running   0          51m   192.168.162.129   kind-worker   <none>           <none>
root@server:~# kubectl delete po -l app.kubernetes.io/instance=nodepinger \
  --field-selector spec.nodeName=$NODE
pod "nodepinger-goldpinger-gx4cj" deleted
root@server:~# kubectl get po -l app.kubernetes.io/instance=nodepinger \
  --field-selector spec.nodeName=$NODE -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
nodepinger-goldpinger-49v79   1/1     Running   0          11s   10.244.2.19   kind-worker   <none>           <none>
root@server:~# curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
goldpinger_nodes_health_total{goldpinger_instance="kind-control-plane",status="healthy"} 5
goldpinger_nodes_health_total{goldpinger_instance="kind-control-plane",status="unhealthy"} 0
root@server:~# cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium-envoy       Desired: 5, Ready: 5/5, Available: 5/5
DaemonSet              cilium             Desired: 5, Ready: 5/5, Available: 5/5
Containers:            cilium             Running: 5
                       cilium-operator    Running: 1
                       cilium-envoy       Running: 5
Cluster Pods:          1/26 managed by Cilium
Helm chart version:    
Image versions         cilium             quay.io/cilium/cilium:v1.16.0@sha256:46ffa4ef3cf6d8885dcc4af5963b0683f7d59daa90d49ed9fb68d3b1627fe058: 5
                       cilium-operator    quay.io/cilium/operator-generic:v1.16.0@sha256:d6621c11c4e4943bf2998af7febe05be5ed6fdcf812b27ad4388f47022190316: 1
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.29.7-39a2a56bbd5b3a591f69dbca51d3e30ef97e0e51@sha256:bd5ff8c66716080028f414ec1cb4f7dc66f40d2fb5a009fff187f4a9b90b566b: 5
root@server:~# kubectl get ciliumnode kind-worker \
  -o jsonpath='{.spec.ipam.podCIDRs[0]}{"\n"}'
10.244.2.0/24
root@server:~# kubectl uncordon $NODE
node/kind-worker uncordoned
root@server:~# kubectl scale deployment goldpinger --replicas 15
deployment.apps/goldpinger scaled
root@server:~# kubectl get po -l app=goldpinger --field-selector spec.nodeName=$NODE -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
goldpinger-57dfcb7b86-4mc48   1/1     Running   0          8s    10.244.2.117   kind-worker   <none>           <none>
goldpinger-57dfcb7b86-74xpq   1/1     Running   0          8s    10.244.2.47    kind-worker   <none>           <none>
goldpinger-57dfcb7b86-dn8b9   1/1     Running   0          8s    10.244.2.100   kind-worker   <none>           <none>
root@server:~# curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
goldpinger_nodes_health_total{goldpinger_instance="kind-control-plane",status="healthy"} 10
goldpinger_nodes_health_total{goldpinger_instance="kind-control-plane",status="unhealthy"} 0
root@server:~# for i in $(seq 2 4); do
  node="kind-worker${i}"
  echo "Migrating node ${node}"
  kubectl drain $node --ignore-daemonsets
  kubectl label node $node --overwrite "io.cilium.migration/cilium-default=true"
  kubectl -n kube-system delete pod --field-selector spec.nodeName=$node -l k8s-app=cilium
  kubectl -n kube-system rollout status ds/cilium -w
  kubectl uncordon $node
done
Migrating node kind-worker2
node/kind-worker2 cordoned
Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-f4w4k, calico-system/csi-node-driver-mzvs7, default/nodepinger-goldpinger-jg2f9, kube-system/cilium-envoy-7qzb7, kube-system/cilium-rhvxk, kube-system/kube-proxy-r7fvz
evicting pod kube-system/cilium-operator-676d46f99b-kdklm
evicting pod default/goldpinger-57dfcb7b86-58m2t
evicting pod calico-apiserver/calico-apiserver-65f68d4985-klvnv
evicting pod default/goldpinger-57dfcb7b86-5jw77
evicting pod default/goldpinger-57dfcb7b86-crjjs
evicting pod calico-system/calico-typha-845fcccf46-226wr
pod/cilium-operator-676d46f99b-kdklm evicted
pod/calico-apiserver-65f68d4985-klvnv evicted
pod/goldpinger-57dfcb7b86-crjjs evicted
pod/goldpinger-57dfcb7b86-58m2t evicted
pod/goldpinger-57dfcb7b86-5jw77 evicted
pod/calico-typha-845fcccf46-226wr evicted
node/kind-worker2 drained
node/kind-worker2 labeled
pod "cilium-rhvxk" deleted
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
node/kind-worker2 uncordoned
Migrating node kind-worker3
node/kind-worker3 cordoned
Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-ckwb4, calico-system/csi-node-driver-gbkxw, default/nodepinger-goldpinger-sdrq2, kube-system/cilium-envoy-6sq84, kube-system/cilium-tx74t, kube-system/kube-proxy-wvmtc
evicting pod default/goldpinger-57dfcb7b86-zxd9x
evicting pod default/goldpinger-57dfcb7b86-5nkk4
evicting pod default/goldpinger-57dfcb7b86-854r7
evicting pod calico-system/calico-typha-845fcccf46-594jl
evicting pod default/goldpinger-57dfcb7b86-6brmk
pod/goldpinger-57dfcb7b86-854r7 evicted
pod/goldpinger-57dfcb7b86-5nkk4 evicted
pod/goldpinger-57dfcb7b86-6brmk evicted
pod/goldpinger-57dfcb7b86-zxd9x evicted
pod/calico-typha-845fcccf46-594jl evicted
node/kind-worker3 drained
node/kind-worker3 labeled
pod "cilium-tx74t" deleted
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
node/kind-worker3 uncordoned
Migrating node kind-worker4
node/kind-worker4 cordoned
Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-zgm9q, calico-system/csi-node-driver-ntn6j, default/nodepinger-goldpinger-f8n8q, kube-system/cilium-cxcfg, kube-system/cilium-envoy-zrqcj, kube-system/kube-proxy-jvcng
evicting pod default/goldpinger-57dfcb7b86-s4nmq
evicting pod default/goldpinger-57dfcb7b86-stljl
evicting pod tigera-operator/tigera-operator-774b5dd976-x2qlw
evicting pod default/goldpinger-57dfcb7b86-xk5wd
evicting pod calico-apiserver/calico-apiserver-65f68d4985-56z2p
evicting pod default/goldpinger-57dfcb7b86-tcbcp
pod/calico-apiserver-65f68d4985-56z2p evicted
pod/tigera-operator-774b5dd976-x2qlw evicted
pod/goldpinger-57dfcb7b86-s4nmq evicted
pod/goldpinger-57dfcb7b86-stljl evicted
pod/goldpinger-57dfcb7b86-tcbcp evicted
pod/goldpinger-57dfcb7b86-xk5wd evicted
node/kind-worker4 drained
node/kind-worker4 labeled
pod "cilium-cxcfg" deleted
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
node/kind-worker4 uncordoned
root@server:~# cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium-envoy       Desired: 5, Ready: 5/5, Available: 5/5
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium             Desired: 5, Ready: 5/5, Available: 5/5
Containers:            cilium-operator    Running: 1
                       cilium             Running: 5
                       cilium-envoy       Running: 5
Cluster Pods:          15/31 managed by Cilium
Helm chart version:    
Image versions         cilium-operator    quay.io/cilium/operator-generic:v1.16.0@sha256:d6621c11c4e4943bf2998af7febe05be5ed6fdcf812b27ad4388f47022190316: 1
                       cilium             quay.io/cilium/cilium:v1.16.0@sha256:46ffa4ef3cf6d8885dcc4af5963b0683f7d59daa90d49ed9fb68d3b1627fe058: 5
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.29.7-39a2a56bbd5b3a591f69dbca51d3e30ef97e0e51@sha256:bd5ff8c66716080028f414ec1cb4f7dc66f40d2fb5a009fff187f4a9b90b566b: 5
root@server:~# kubectl rollout restart daemonset nodepinger-goldpinger
kubectl rollout status daemonset nodepinger-goldpinger
daemonset.apps/nodepinger-goldpinger restarted
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 0 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 1 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 1 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 2 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 2 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 3 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 3 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 3 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 4 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 4 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 4 of 5 updated pods are available...
daemon set "nodepinger-goldpinger" successfully rolled out
root@server:~# kubectl get po -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP              NODE                 NOMINATED NODE   READINESS GATES
goldpinger-57dfcb7b86-4mc48   1/1     Running   0          3m18s   10.244.2.117    kind-worker          <none>           <none>
goldpinger-57dfcb7b86-4pnbv   1/1     Running   0          50s     10.244.1.63     kind-worker3         <none>           <none>
goldpinger-57dfcb7b86-5wtkq   1/1     Running   0          50s     10.244.1.253    kind-worker3         <none>           <none>
goldpinger-57dfcb7b86-5xz69   1/1     Running   0          80s     10.244.3.224    kind-worker2         <none>           <none>
goldpinger-57dfcb7b86-74xpq   1/1     Running   0          3m18s   10.244.2.47     kind-worker          <none>           <none>
goldpinger-57dfcb7b86-7wkq2   1/1     Running   0          80s     10.244.3.179    kind-worker2         <none>           <none>
goldpinger-57dfcb7b86-bf6z6   1/1     Running   0          48m     192.168.82.7    kind-control-plane   <none>           <none>
goldpinger-57dfcb7b86-dn8b9   1/1     Running   0          3m18s   10.244.2.100    kind-worker          <none>           <none>
goldpinger-57dfcb7b86-kqnsp   1/1     Running   0          3m18s   192.168.82.9    kind-control-plane   <none>           <none>
goldpinger-57dfcb7b86-npmlp   1/1     Running   0          50s     10.244.1.240    kind-worker3         <none>           <none>
goldpinger-57dfcb7b86-qlc7j   1/1     Running   0          48m     192.168.82.8    kind-control-plane   <none>           <none>
goldpinger-57dfcb7b86-rtppl   1/1     Running   0          50s     10.244.1.1      kind-worker3         <none>           <none>
goldpinger-57dfcb7b86-tr6gg   1/1     Running   0          80s     10.244.3.234    kind-worker2         <none>           <none>
goldpinger-57dfcb7b86-vz944   1/1     Running   0          103s    10.244.2.209    kind-worker          <none>           <none>
goldpinger-57dfcb7b86-w574c   1/1     Running   0          80s     10.244.3.163    kind-worker2         <none>           <none>
nodepinger-goldpinger-4mtlr   1/1     Running   0          7s      10.244.4.220    kind-worker4         <none>           <none>
nodepinger-goldpinger-762hs   1/1     Running   0          6s      192.168.82.10   kind-control-plane   <none>           <none>
nodepinger-goldpinger-77gc8   1/1     Running   0          8s      10.244.2.191    kind-worker          <none>           <none>
nodepinger-goldpinger-r6j8p   1/1     Running   0          5s      10.244.1.192    kind-worker3         <none>           <none>
nodepinger-goldpinger-x5944   1/1     Running   0          10s     10.244.3.231    kind-worker2         <none>           <none>
root@server:~# NODE="kind-control-plane"
kubectl drain $NODE --ignore-daemonsets
kubectl label node $NODE --overwrite "io.cilium.migration/cilium-default=true"
kubectl -n kube-system delete pod --field-selector spec.nodeName=$NODE -l k8s-app=cilium
kubectl -n kube-system rollout status ds/cilium -w
kubectl uncordon $NODE
node/kind-control-plane cordoned
Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-mbjbf, calico-system/csi-node-driver-cs2m6, default/nodepinger-goldpinger-762hs, kube-system/cilium-bf4mv, kube-system/cilium-envoy-c6mlf, kube-system/kube-proxy-mzs2j
evicting pod local-path-storage/local-path-provisioner-7577fdbbfb-s8tbf
evicting pod calico-system/calico-kube-controllers-85955d4f5b-5j7hr
evicting pod kube-system/coredns-76f75df574-g9nq4
evicting pod default/goldpinger-57dfcb7b86-bf6z6
evicting pod default/goldpinger-57dfcb7b86-qlc7j
evicting pod kube-system/coredns-76f75df574-x7tl4
evicting pod default/goldpinger-57dfcb7b86-kqnsp
pod/goldpinger-57dfcb7b86-kqnsp evicted
pod/calico-kube-controllers-85955d4f5b-5j7hr evicted
pod/goldpinger-57dfcb7b86-bf6z6 evicted
pod/goldpinger-57dfcb7b86-qlc7j evicted
pod/coredns-76f75df574-x7tl4 evicted
pod/coredns-76f75df574-g9nq4 evicted
pod/local-path-provisioner-7577fdbbfb-s8tbf evicted
node/kind-control-plane drained
node/kind-control-plane labeled
pod "cilium-bf4mv" deleted
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
node/kind-control-plane uncordoned
root@server:~# kubectl rollout restart daemonset nodepinger-goldpinger
kubectl rollout status daemonset nodepinger-goldpinger
daemonset.apps/nodepinger-goldpinger restarted
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 0 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 1 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 1 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 1 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 2 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 2 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 3 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 3 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 3 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 4 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 4 out of 5 new pods have been updated...
Waiting for daemon set "nodepinger-goldpinger" rollout to finish: 4 of 5 updated pods are available...
daemon set "nodepinger-goldpinger" successfully rolled out
root@server:~# kubectl rollout restart daemonset -n calico-system csi-node-driver
kubectl rollout status daemonset -n calico-system csi-node-driver
daemonset.apps/csi-node-driver restarted
Waiting for daemon set "csi-node-driver" rollout to finish: 0 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 0 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 1 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 1 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 2 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 2 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 3 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 3 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 4 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 4 out of 5 new pods have been updated...
Waiting for daemon set "csi-node-driver" rollout to finish: 4 of 5 updated pods are available...
daemon set "csi-node-driver" successfully rolled out
root@server:~# cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium-envoy       Desired: 5, Ready: 5/5, Available: 5/5
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium             Desired: 5, Ready: 5/5, Available: 5/5
Containers:            cilium             Running: 5
                       cilium-envoy       Running: 5
                       cilium-operator    Running: 1
Cluster Pods:          31/31 managed by Cilium
Helm chart version:    
Image versions         cilium             quay.io/cilium/cilium:v1.16.0@sha256:46ffa4ef3cf6d8885dcc4af5963b0683f7d59daa90d49ed9fb68d3b1627fe058: 5
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.29.7-39a2a56bbd5b3a591f69dbca51d3e30ef97e0e51@sha256:bd5ff8c66716080028f414ec1cb4f7dc66f40d2fb5a009fff187f4a9b90b566b: 5
                       cilium-operator    quay.io/cilium/operator-generic:v1.16.0@sha256:d6621c11c4e4943bf2998af7febe05be5ed6fdcf812b27ad4388f47022190316: 1
root@server:~# 
```

## 🧹 Clean-up
Now the migration has been completed, let's update the Cilium configuration to support Network Policies and remove the previous network plugin.

### Update the cilium configuration

Now that Cilium is healthy, let's update the Cilium configuration.

First, generate a new Helm values file by overriding some parameters:

shell

copy

run

```bash
cilium install \
  --helm-values values-initial.yaml \
  --helm-set operator.unmanagedPodWatcher.restart=true \
  --helm-set cni.customConf=false \
  --helm-set policyEnforcementMode=default \
  --dry-run-helm-values > values-final.yaml
```

Again, we are using the cilium-cli to generate an updated Helm config file. Check the differences with the previous values:

shell

copy

run

```bash
diff -u --color values-initial.yaml values-final.yaml
```

As you can see from checking the differences between the two files, we are changing three parameters:

enabling Cilium to write the CNI configuration file by disabling a per node configuration (customConf)
enabling Cilium to restart unmanaged pods
enabling Network Policy Enforcement
Let's apply it and rollout the Cilium DaemonSet:

shell

copy

run

```bash
helm upgrade --install \
  --namespace kube-system cilium cilium/cilium \
  --values values-final.yaml
kubectl -n kube-system rollout restart daemonset cilium
cilium status --wait
```

Remove the CiliumNodeConfig resource:

shell

copy

run
kubectl delete -n kube-system ciliumnodeconfig cilium-default

### Delete the previous network plugin

Let's remove Calico as it is no longer needed:

shell

copy

run

```bash
kubectl delete --force -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
kubectl delete --force -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
kubectl delete --force namespace calico-system
```

If the commands block, stop them with Ctrl+C and try again.

### Restart the Nodes

Finally, let's restart the nodes one by one:

shell

copy

run

```bash
for i in " " $(seq 1 4); do
  node="kind-worker${i}"
  echo "Restarting node ${node}"
  docker restart $node
  sleep 5 # wait for cilium to catch that the node is missing
  kubectl -n kube-system rollout status ds/cilium -w
done
```

Let's finish with the control plane node:

shell

copy

run

```bash
docker restart kind-control-plane
sleep 5
kubectl -n kube-system rollout status ds/cilium -w
```

Check Cilium status:

shell

copy

run

```bash
cilium status --wait
```

All pods should now be managed by Cilium (see the "Cluster Pods" section).

Check that connectivity is still fine for both the DaemonSet and the Deployment Goldpinger apps:

shell

copy

run

```bash
curl -s http://localhost:32042/metrics | grep '^goldpinger_nodes_health_total'
curl -s http://localhost:32043/metrics | grep '^goldpinger_nodes_health_total'
```

Since we've restarted the only control plane in this cluster (you should have at least 3 in production clusters), the state might be a bit broken for a little while.

Check the connectivity:

shell

copy

run

```bash
cilium connectivity test
```

We've now successfully migrated to Cilium!

#### Console Output

```bash
root@server:~# cilium install \
  --helm-values values-initial.yaml \
  --helm-set operator.unmanagedPodWatcher.restart=true \
  --helm-set cni.customConf=false \
  --helm-set policyEnforcementMode=default \
  --dry-run-helm-values > values-final.yaml
root@server:~# diff -u --color values-initial.yaml values-final.yaml
--- values-initial.yaml 2024-08-10 13:18:01.459036823 +0000
+++ values-final.yaml   2024-08-10 13:49:45.815275652 +0000
@@ -3,7 +3,7 @@
 cluster:
   name: kind-kind
 cni:
-  customConf: true
+  customConf: false
   uninstall: false
 ipam:
   mode: cluster-pool
@@ -13,8 +13,8 @@
 operator:
   replicas: 1
   unmanagedPodWatcher:
-    restart: false
-policyEnforcementMode: never
+    restart: true
+policyEnforcementMode: default
 routingMode: tunnel
 tunnel: vxlan
 tunnelPort: 8473
root@server:~# yq values-final.yaml
bpf:
  hostLegacyRouting: true
cluster:
  name: kind-kind
cni:
  customConf: false
  uninstall: false
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - 10.244.0.0/16
operator:
  replicas: 1
  unmanagedPodWatcher:
    restart: true
policyEnforcementMode: default
routingMode: tunnel
tunnel: vxlan
tunnelPort: 8473
tunnelProtocol: vxlan
root@server:~# helm upgrade --install \
  --namespace kube-system cilium cilium/cilium \
  --values values-final.yaml
kubectl -n kube-system rollout restart daemonset cilium
cilium status --wait
Release "cilium" has been upgraded. Happy Helming!
NAME: cilium
LAST DEPLOYED: Sat Aug 10 13:51:18 2024
NAMESPACE: kube-system
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
You have successfully installed Cilium with Hubble.

Your release version is 1.16.0.

For any further help, visit https://docs.cilium.io/en/v1.16/gettinghelp
daemonset.apps/cilium restarted
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium-envoy       Desired: 5, Ready: 5/5, Available: 5/5
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium             Desired: 5, Ready: 5/5, Available: 5/5
Containers:            cilium             Running: 5
                       cilium-envoy       Running: 5
                       cilium-operator    Running: 1
Cluster Pods:          31/31 managed by Cilium
Helm chart version:    
Image versions         cilium             quay.io/cilium/cilium:v1.16.0@sha256:46ffa4ef3cf6d8885dcc4af5963b0683f7d59daa90d49ed9fb68d3b1627fe058: 5
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.29.7-39a2a56bbd5b3a591f69dbca51d3e30ef97e0e51@sha256:bd5ff8c66716080028f414ec1cb4f7dc66f40d2fb5a009fff187f4a9b90b566b: 5
                       cilium-operator    quay.io/cilium/operator-generic:v1.16.0@sha256:d6621c11c4e4943bf2998af7febe05be5ed6fdcf812b27ad4388f47022190316: 1
root@server:~# kubectl delete -n kube-system ciliumnodeconfig cilium-default
ciliumnodeconfig.cilium.io "cilium-default" deleted
root@server:~# kubectl delete --force -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
kubectl delete --force -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
kubectl delete --force namespace calico-system
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
namespace "tigera-operator" force deleted
customresourcedefinition.apiextensions.k8s.io "bgpconfigurations.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "bgppeers.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "blockaffinities.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "caliconodestatuses.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "clusterinformations.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "felixconfigurations.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "globalnetworkpolicies.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "globalnetworksets.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "hostendpoints.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "ipamblocks.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "ipamconfigs.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "ipamhandles.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "ippools.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "ipreservations.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "kubecontrollersconfigurations.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "networkpolicies.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "networksets.crd.projectcalico.org" force deleted
customresourcedefinition.apiextensions.k8s.io "apiservers.operator.tigera.io" force deleted
customresourcedefinition.apiextensions.k8s.io "imagesets.operator.tigera.io" force deleted
customresourcedefinition.apiextensions.k8s.io "installations.operator.tigera.io" force deleted
customresourcedefinition.apiextensions.k8s.io "tigerastatuses.operator.tigera.io" force deleted
serviceaccount "tigera-operator" force deleted
clusterrole.rbac.authorization.k8s.io "tigera-operator" force deleted
clusterrolebinding.rbac.authorization.k8s.io "tigera-operator" force deleted
deployment.apps "tigera-operator" force deleted

^CWarning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
installation.operator.tigera.io "default" force deleted
Error from server (NotFound): error when deleting "https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml": the server could not find the requested resource (delete apiservers.operator.tigera.io default)
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
namespace "calico-system" force deleted
root@server:~# for i in " " $(seq 1 4); dodo
  node="kind-worker${i}"
  echo "Restarting node ${node}"
  docker restart $node
  sleep 5 # wait for cilium to catch that the node is missing
  kubectl -n kube-system rollout status ds/cilium -w
done
Restarting node kind-worker 
kind-worker
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
Restarting node kind-worker1
Error response from daemon: No such container: kind-worker1
daemon set "cilium" successfully rolled out
Restarting node kind-worker2
kind-worker2
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
Restarting node kind-worker3
kind-worker3
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
Restarting node kind-worker4
kind-worker4
Waiting for daemon set "cilium" rollout to finish: 4 of 5 updated pods are available...
daemon set "cilium" successfully rolled out
root@server:~# docker restart kind-control-plane
sleep 5
kubectl -n kube-system rollout status ds/cilium -w
kind-control-plane
daemon set "cilium" successfully rolled out
root@server:~# cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium-envoy       Desired: 5, Ready: 5/5, Available: 5/5
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium             Desired: 5, Ready: 5/5, Available: 5/5
Containers:            cilium             Running: 5
                       cilium-envoy       Running: 5
                       cilium-operator    Running: 1
Cluster Pods:          23/23 managed by Cilium
Helm chart version:    
Image versions         cilium-envoy       quay.io/cilium/cilium-envoy:v1.29.7-39a2a56bbd5b3a591f69dbca51d3e30ef97e0e51@sha256:bd5ff8c66716080028f414ec1cb4f7dc66f40d2fb5a009fff187f4a9b90b566b: 5
                       cilium-operator    quay.io/cilium/operator-generic:v1.16.0@sha256:d6621c11c4e4943bf2998af7febe05be5ed6fdcf812b27ad4388f47022190316: 1
                       cilium             quay.io/cilium/cilium:v1.16.0@sha256:46ffa4ef3cf6d8885dcc4af5963b0683f7d59daa90d49ed9fb68d3b1627fe058: 5
root@server:~# 





```


