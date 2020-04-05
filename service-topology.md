## Service topology, route local!

### Introduction 
The service topology is a new feature available in Kubernetes v1.17. This essentially lets your route traffic based on the topology of your clusters. This gives you a more granular approach on where you want traffic to end up in the cluster. Say you want to route service traffic to the same node as the client or the same availability zone each time then this is what you can use to define.

### Enabling

There are a few prerequisites you need to ensure are on your cluster.
The cluster needs to be running v1.17 hence it’s a new feature
Need to have kube-proxy running as either IPtables or IPVS mode
Need to enable [Endpoint Slices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) 

To enable just add the feature gate.

```--feature-gates="ServiceTopology=true,EndpointSlice=true"```

This needs to be enabled on all components where feature gates are supported.

If you caught wondering what endpoint slice is, this was introduced in Kubernetes v.1.16. The Topology aware service routing will use this to implement endpoint filtering and then convert to iptables or ipvs rules to implement nearby forwarding.

### Configuration

The main config is to add ```topologyKeys``` in your service spec. This act as a preference-order list of node labels used to tell which endpoint traffic will be routed too. The value is related to the current node.
Traffic will be directed based on the keys set and will go through the list trying to match each key if not it will try next key until all options have been exhausted. Think of this as a network access control list-style approach.
If` ```topologyKeys``` is not specified or left blank no filtering will happen. 
One thing to take note of If no matches are found in list traffic will be dropped as if there where no backend endpoints at all. To combat this we can place a wildcard value at the end of every list ‘’’*’’’ which means any topology and can be used as a catch-all. This catch-all value can only be placed last in the list nowhere else. 

### What keys can I use?

Currently, as of now, there are only the following keys that can be set. 

```kubernetes.io/hostname``` = This represents the hostname of the node. So if the local machine has an endpoint it will be forwarded to directly to it.

```topology.kubernetes.io/zone``` = This represents the availability zone of where the node is located, traffic will be forward to a node in this zone.

```topology.kubernetes.io/region``` = This represents the region of where the node is located. Although in some rare cases you might have clusters with cross-region nodes, latency can be very high.
```*```= The wild card value to match all endpoints.
You can set these keys in a different order for your specific use case.

### In action!
So let’s take a look at the example below, we configured our service as the following.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  topologyKeys:
  - "kubernetes.io/hostname"
  - "topology.kubernetes.io/zone"
  - "topology.kubernetes.io/region"
  - "*"
```

This will traverse as following. 
1. Use endpoint on the host if not,
2. Use endpoint in the availability zone if not,
3. Use endpoint in the region if not, 
4. Use any endpoint at random.

![image of service-topology](/images/service-topology/service-topology.png)

Now you have seen the concept in more detail let's follow along with the demo below. This has been configured on a 3 nodes cluster using `kind` (kind is a great tool that runs docker containers as kubernetes nodes and is great for testing new features!).

Lets see what our test cluster looks like
```
kubectl get nodes -o wide
NAME                 STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
kind-control-plane   Ready    master   29m   v1.17.2   172.17.0.3    <none>        Ubuntu 19.10   4.19.76-linuxkit   containerd://1.3.2-31-gaa877d78
kind-worker          Ready    <none>   29m   v1.17.2   172.17.0.2    <none>        Ubuntu 19.10   4.19.76-linuxkit   containerd://1.3.2-31-gaa877d78
kind-worker2         Ready    <none>   29m   v1.17.2   172.17.0.4    <none>        Ubuntu 19.10   4.19.76-linuxkit   containerd://1.3.2-31-gaa877d78
```
Take note of the internal-IP set for each nodes.

We have a deployment of nginx running with 5 pods.
``` kubectl get deployment
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   5/5     5            5           29m
```
```
kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
nginx-59f986c477-4flws   1/1     Running   0          23m   10.244.2.4   kind-worker2   <none>           <none>
nginx-59f986c477-8w2h7   1/1     Running   0          23m   10.244.1.6   kind-worker    <none>           <none>
nginx-59f986c477-lb5h7   1/1     Running   0          23m   10.244.1.5   kind-worker    <none>           <none>
nginx-59f986c477-qt6rv   1/1     Running   0          23m   10.244.2.5   kind-worker2   <none>           <none>
nginx-59f986c477-rmcpl   1/1     Running   0          23m   10.244.2.6   kind-worker2   <none>           <none>
```
We can see that pods are distrubuted over both worker nodes.

Now let's see the service exposing that deployment and its configuration.
```
kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        39m
nginx        NodePort    10.98.152.251   <none>        80:31206/TCP   30m
```
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  type: NodePort
  topologyKeys:
   - "kubernetes.io/hostname"
   - "topology.kubernetes.io/zone"
   - "*"
status:
  loadBalancer: {}
```
We have a node port service with three topologyKeys set.
Once service is configured and created we can look at the endpointslice for that service.
```
kubectl get endpointslice
NAME          ADDRESSTYPE   PORTS   ENDPOINTS                                      AGE
kubernetes    IPv4          6443    172.17.0.3                                     42m
nginx-w2n4v   IPv4          80      10.244.1.5,10.244.2.6,10.244.2.4 + 2 more...   33m
```
```
kubectl describe endpointslice nginx-w2n4v
Name:         nginx-w2n4v
Namespace:    default
Labels:       endpointslice.kubernetes.io/managed-by=endpointslice-controller.k8s.io
              kubernetes.io/service-name=nginx
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2020-04-05T10:40:23Z
AddressType:  IPv4
Ports:
  Name     Port  Protocol
  ----     ----  --------
  <unset>  80    TCP
Endpoints:
  - Addresses:  10.244.1.5
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/nginx-59f986c477-lb5h7
    Topology:   kubernetes.io/hostname=kind-worker
                topology.kubernetes.io/zone=AZ-A
  - Addresses:  10.244.2.6
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/nginx-59f986c477-rmcpl
    Topology:   kubernetes.io/hostname=kind-worker2
                topology.kubernetes.io/zone=AZ-B
  - Addresses:  10.244.2.4
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/nginx-59f986c477-4flws
    Topology:   kubernetes.io/hostname=kind-worker2
                topology.kubernetes.io/zone=AZ-B
  - Addresses:  10.244.2.5
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/nginx-59f986c477-qt6rv
    Topology:   kubernetes.io/hostname=kind-worker2
                topology.kubernetes.io/zone=AZ-B
  - Addresses:  10.244.1.6
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/nginx-59f986c477-8w2h7
    Topology:   kubernetes.io/hostname=kind-worker
                topology.kubernetes.io/zone=AZ-A
Events:         <none>
```
Here we can see listed all endpoints for that service with there corresponding topology keys set. These have also been set as node labels on each node.
So according to this current configuration if external traffic entering the cluster hits node `kind-worker` it should always use local pods on node ending `lb5h7` `8w2h7`
If external traffic entering cluster hits node `kind-worker2` it should always use local pods on node ending `rmcpl` `4flws` `qt6rv`
If external traffic entering cluster hits node `kind-control-plane` it has no endpoints listed locally so it will use next key in secqunce which is zone. Lets see what we have labelled as zone on `kind-control-plane` node.
```
kubectl describe nodes kind-control-plane
Name:               kind-control-plane
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=kind-control-plane
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
                    topology.kubernetes.io/zone=AZ-A
```
As we have label `topology.kubernetes.io/zone=AZ-A` set it will use the same endpoints as `kind-worker`

Now lets generate some external traffic and see if this actually works. We have set up a seperate node oustide of the cluster to simulate external traffic and we will use wget to hit the node ip attached with service node port.
Let's run below command hitting node `kind-worker` on it's internal IP with correct node port for our nginx service.
```
/ # for i in `seq 0 99`;do wget -q http://172.17.0.2:31206 -O - |grep '<h1>Welcome'; done | sort | uniq -c
     56 <h1>Welcome to nginx! nginx-59f986c477-8w2h7</h1>
     44 <h1>Welcome to nginx! nginx-59f986c477-lb5h7</h1>
```
As you can see it hits both endpoints we predicted earlier.

Running same command hitting node `kind-worker2`
```
/ # for i in `seq 0 99`;do wget -q http://172.17.0.4:31206 -O - |grep '<h1>Welcome'; done | sort | uniq -c
     34 <h1>Welcome to nginx! nginx-59f986c477-4flws</h1>
     34 <h1>Welcome to nginx! nginx-59f986c477-qt6rv</h1>
     32 <h1>Welcome to nginx! nginx-59f986c477-rmcpl</h1>
```
As you can see its hits the three endpoints we predicted earlier.

Finally lets run command on node `kind-control-plane`
```
/ # for i in `seq 0 99`;do wget -q http://172.17.0.3:31206 -O - |grep '<h1>Welcome'; done | sort | uniq -c
     48 <h1>Welcome to nginx! nginx-59f986c477-8w2h7</h1>
     52 <h1>Welcome to nginx! nginx-59f986c477-lb5h7</h1>
```
As you can see it's the same results as `kind-worker` but in this case it's using the key `topology.kubernetes.io/zone` and both nodes happen to be in same availability zone.

So what would happen if we removed this key `topology.kubernetes.io/zone` from our service configuration. What would happen with traffic hitting `kind-control-plane` node. Let's test and see!

New service config file with removed zone topology key.
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  type: NodePort
  topologyKeys:
   - "kubernetes.io/hostname"
   - "*"
status:
  loadBalancer: {}
```
Let's delete service,redeploy and run same command.
```
/ # for i in `seq 0 99`;do wget -q http://172.17.0.3:31206 -O - |grep '<h1>Welcome'; done | sort | uniq -c
     11 <h1>Welcome to nginx! nginx-59f986c477-4flws</h1>
     30 <h1>Welcome to nginx! nginx-59f986c477-8w2h7</h1>
     17 <h1>Welcome to nginx! nginx-59f986c477-lb5h7</h1>
     14 <h1>Welcome to nginx! nginx-59f986c477-qt6rv</h1>
     28 <h1>Welcome to nginx! nginx-59f986c477-rmcpl</h1>
```
We can now see traffic is hitting all endpoints. As there are no endpoints locally on host and we have removed zone topology key, traffic will now use wild card `*` key inturn hitting all endpoints binded to that service.

### Summary
Topology-aware service routing is a great way to control the use of only local backends defining service forwarding nearby which in turn can reduce latency, improve security/performance and some cases save money. Hopefully, there will be more topology keys for use in the future as this is still a new feature.

