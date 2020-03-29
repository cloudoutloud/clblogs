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


### Summary
Topology-aware service routing is a great way to control the use of only local backends defining service forwarding nearby which in turn can reduce latency, improve security/performance and some case save money. Hopefully, there will be more topology keys for use in the future as this is still a new feature.

