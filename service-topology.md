## Service topology, route to where you want!


### introduction 
The service topology is a new feature available in Kubernetes v1.17. This essentially lets your route traffic based on the topology of your clusters. This gives you a more granular approach on where you want traffic to end up in the cluster. Say you want to route service traffic to the same node as the client or the same availability zone each time then this is what you can use to define.

<pic Diagram of service topology overview>


### Enabling

There are a few prerequisites you need to ensure are on your cluster.
- The cluster needs to be running v1.17 hence it’s a new feature
- Need to have kube-proxy running as either IPtables or IPVS mode
- Need to enable [Endpoint Slices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)

To enable service topology feature just add.
```--feature-gates="ServiceTopology=true```


### Configuration

The main config is to add ```topologyKeys``` in your service spec. This act as a preference-order list of node labels used to tell which endpoint traffic will be routed too. 
Traffic will be directed < explain how this goes through list >
If` ```topologyKeys``` is not specified or left blank no filtering will happen. 
One thing to take note of If no matches are found in list traffic will be dropped as if there where no backend endpoints at all. To combat this we can place a wildcard value at the end of every list ```*``` which means any topology and can be used as a catch-all. This catch-all value can only be placed last in the list nowhere else. 


### In action




### Summary
