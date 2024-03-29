## Running Kube-proxy in super mode!

### The modes of Kube-proxy
First off Kube-proxy is a network proxy that runs as a daemonset in the kube-system namespace this ensures at least one pod is running on all nodes in the cluster.
Kube-proxy's main job is to load balance traffic that is destined for services via (cluster IP’s and node port) to the correct backend pods. The current modes Kube-proxy can run in are IPtables (most common and default mode) and IPVS. There is also a third mode userspace, but this is very old and slow so please do not use it! and we will not mention.

### IPtables mode
So let's have a recap of the most standard mode most clusters run Kube-proxy as.
IPtables is a Linux kernel feature designed to be an efficient firewall. The Kube-proxy programs the IPtables rules means that it is nominally an O(n) style algorithm, where n grows roughly in proportion to your cluster size. So it scales to accommodate the number of services and it’s backend pods. It uses a randomized equal-cost selection algorithm. The downside is IPtables struggles to scale as cluster grow larger as it's designed purely as a firewall with a rules list chain approach. For example, Kubernetes v.1.17 can support up to 5000 nodes but running Iptables could become a nasty bottleneck.

### IPVS mode to the rescue
IPVS which stands for “IP Virtual Server”. It was introduced way back in Kubernetes v1.11 release but it's often overlooked and underutilized.
If you have 1000’s of services and large node clusters it would be more efficient to use IPVS mode over IPtables. IPVS is a mature Linux feature specifically designed for load balancing lots of services. It has an API and a lookup routine rather than an iptables list of sequential rules approach. Request routing is done in kernel space so super fast.
Has a nominal computational complexity of O(1). Its connection processing performance will stay constant independent of your cluster size. Uses more efficient data structures (hash tables) allowing for almost unlimited scale under the hood and can use different load balancing methods default is good old round-robin.

![image of kube-ipvs-mode](/images/ipvs-mode/kube-in-ipvs.png)

### Pros of using IPVS 
- Purley designed for load balancing.
- Based on faster performing in-kernel hash tables.
- Can help kube-proxy to improve cluster performance, when a cluster has 1000’s of services.
- Can support a rich set of connection scheduling algorithms for load balancing.
- Very learn from a resourcing perspective uses a fraction of memory IPtables uses and no CPU.

### Cons of using IPVS 
- Cannot rewrite destination as packet cannot be amended.
- As-built for load balancing cannot handle other kube-proxy workarounds such as packet filtering, hairpin-masquerade tricks and SNAT.
- If you plan to use IPVS with other programs that use iptables your need to test if they behave as expected together as the routing approach is different.

### Enabling and configuration

If running managed services of Kubernetes such as AKS, EKS and GKE. Enabling and configuration may be slightly different. Please consult vendor documentation.
Below is how you can enable and configure on a fully managed cluster using `kubeadm` to provision. 

To enable on a new kubeadm cluster from scratch.

You can add the below to your kubeadm configuration file.

```yaml
...
kubeProxy:
  config:
    featureGates:
      SupportIPVSProxyMode: true
    mode: ipvs
...
```
Pass the file with config flag when using the kubeadm init command.
```kubeadm init --config=```

Alternatively you can set flag below when running kubeadm init command.

```kubeadm init --feature-gates=SupportIPVSProxyMode=true```


To enable IPVS mode on an existing cluster that is running IPtables.

Edit the existing configmap configuration
```kubectl edit configmap kube-proxy -n kube-system```

Change mode under kubeproxy configuation
```mode: ipvs```

Delete kube-proxy pods and daemonset will recreate.

```kubectl get po -n kube-system```
```kubectl delete po -n kube-system <pod-name>```

Check logs of kube-proxy pods to confirm IPVS mode is running.

```kubectl logs [kube-proxy pod]```

```k logs kube-proxy-gqfjz -n kube-system```
```I0308 10:30:50.986140       1 node.go:135] Successfully retrieved node IP: 10.0.2.15
I0308 10:30:50.986187       1 server_others.go:172] Using ipvs Proxier.
W0308 10:30:50.986556       1 proxier.go:414] clusterCIDR not specified, unable to distinguish between internal and external traffic
W0308 10:30:50.986573       1 proxier.go:420] IPVS scheduler not specified, use rr by default
```

As you can see also as we have not set an IPVS scheduler it will use round-robin default.
As mentioned earlier we have a few connection scheduling algorithms for load balancing to choose from.

To switch load balancing methods 
Set  ```--ipvs-scheduler=``` as a kube-proxy parameter or edit the kube-proxy config map.
```kubectl -n kube-system edit configmap kube-proxy```

Full list of options below. Information [here](https://www.keepalived.org/doc/scheduling_algorithms.html)
```
rr: round-robin
lc: least connection
dh: destination hashing
sh: source hashing
sed: shortest expected delay
nq: never queue
```
Optionally you can use ipvsadm CLI tool to interact with IP virtual server in the kernel. Install using, distro package manager such as ```yum install ipvsadm```

### Demo Time!

In this short demo, we will create a deployment and expose as a service. Then load balance to the backend endpoint pods using IPVS. We have the cluster already set up using IPVS mode with 1x master and 2x worker nodes.
Let’s verify the IPVS config using the command on each node. This will show `rr` for round-robin.

```ipvsadm -l```

```IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  ip-10-96-0-1.us-east-2.compu rr
  -> ip-172-31-8-196.us-east-2.co Masq    1      5          0
TCP  ip-10-96-0-10.us-east-2.comp rr
  -> ip-10-32-0-2.us-east-2.compu Masq    1      0          0
  -> ip-10-32-0-3.us-east-2.compu Masq    1      0          0
TCP  ip-10-96-0-10.us-east-2.comp rr
  -> ip-10-32-0-2.us-east-2.compu Masq    1      0          0
  -> ip-10-32-0-3.us-east-2.compu Masq    1      0          0
UDP  ip-10-96-0-10.us-east-2.comp rr
  -> ip-10-32-0-2.us-east-2.compu Masq    1      0          0
  -> ip-10-32-0-3.us-east-2.compu Masq    1      0          0
  ```

Now let’s create the deployment using image `whats-my-ip`

```kubectl run myip --image=cloudnativelabs/whats-my-ip --replicas=2 --port=8080```

Two pods have been deployed across the two worker nodes.

`10.44.0.1` Endpoint for kube-worker1
`10.36.0.1` Endpoint for kube-worker2

```
NAME                   READY   STATUS    RESTARTS   AGE   IP          NODE           NOMINATED NODE   READINESS GATES
myip-b4ffc5c6f-6l2hc   1/1     Running   0          11s   10.36.0.1   kube-worker2   <none>           <none>
myip-b4ffc5c6f-dwb7f   1/1     Running   0          11s   10.44.0.1   kube-worker1   <none>           <none>
```

Expose the deployment as a service (nodeport)
```kubectl expose deployment myip --port=8080 --target-port=8080 --type=NodePort```

We can see svc was created and the endpoints listed
```kubectl get svc```
```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP          35m
myip         NodePort    10.98.98.178   <none>        8080:31579/TCP   9s
```

```kubectl get ep -o wide```
```
NAME         ENDPOINTS                       AGE
kubernetes   172.31.8.196:6443               35m
myip         10.36.0.1:8080,10.44.0.1:8080   27s
```

We can now access the svc by running a curl command below to the node IP. This will return information back from application.

```curl <node ip>:<node port>```

So from the master node if we curl worker node 2 IP address on service node port we can see this is load balancing between both endpoints in a round-robin fashion.

```
[centos@kube-master ~]$ curl 172.31.35.73:31579
HOSTNAME:myip-b4ffc5c6f-dwb7f IP:10.44.0.1
[centos@kube-master ~]$ curl 172.31.35.73:31579
HOSTNAME:myip-b4ffc5c6f-6l2hc IP:10.36.0.1
[centos@kube-master ~]$ curl 172.31.35.73:31579
HOSTNAME:myip-b4ffc5c6f-dwb7f IP:10.44.0.1
[centos@kube-master ~]$ curl 172.31.35.73:31579
HOSTNAME:myip-b4ffc5c6f-6l2hc IP:10.36.0.1
```




### Conclusion 

If you do find yourself running large clusters with over 1000 services. IPVS mode is the way to go in order to get the efficiency and performance you require. Even if you are not running huge amounts of services it may be still worth testing as IPVS may become the defacto standard in future Kubernetes releases.

### Links 
[ipvsadm (man-page)](https://linux.die.net/man/8/ipvsadm)

[IP Virtual Server (wikipedia)](https://en.wikipedia.org/wiki/IP_Virtual_Server)

[kube-proxy(Kuberenets docs)](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)

[kubeadm(offical docs)](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
)


