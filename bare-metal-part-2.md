##Kubernetes running on the bare metal public cloud platform Packet!

### Part 2
In this part, we will look at how easy it is to use popular open-source tools to achieve load balancing and service mesh networking within a bare metal Kubernetes cluster.

---

### MetalLB

When you deploy a load balancer service in Kubernetes what usually happens is a load balancer resources will get created in your public cloud provider where your cluster is hosted. In AWS, an elastic load balancer will be created in the cloud with an assigned public IP.

When running your cluster bare metal you will need to essentially create a load balancer resources yourself the easiest way to get this up and running is by using an open-source tool called [MetalLB](https://metallb.universe.tf/)

Once installed MetalLB will create a namespace MetalLB-system in which it will deploy one deployment “controller” which jobs are to handle IP assignment. Also, it will deploy a daemon set “speaker” this is the component that speaks the protocol we require in our case BGP (border gateway protocol) this will make services reachable.

To configure BGP we will create a config map with the following information.

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    peers:
    - peer-address: 10.0.0.1
      peer-asn: 64501
      my-asn: 64500
    address-pools:
    - name: default
      protocol: bgp
      addresses:
      - 192.168.10.0/24
```

Peer-address will be the IP we want to peer with, Peer-address is the peer router address of each node. We can add one peer per a Kubernetes node to include the whole cluster in the BGP configuration. Peer-asn and my-asn needs to be different. Under the addresses section, we can specify an address block. Using Packet we can assign a group of public IP to use with our metalLB this can be done within the console by going to the IP & network tab and clicking request IP address tab. Once we have a group of IPs to use and specified in config map metal LB will then allocate them to any newly created load balancer services within our cluster.

Using metalLB we can make use of BGP to route traffic between nodes and have seamless failover when one node dies or become faulty. Traffic will seamlessly switch within seconds to the operational node there are only a few packets dropped. There is also an option to load balance between nodes. Within Packet UI we need to enable BGP within the network tab to make use of these features.

---

### Istio

If you have heard of service mesh within a network you have probably heard of a tool called [Istio](https://istio.io/). Istio is a complex tool and there are many posts and documents explaining it’s different features and services I won’t dive into this too much. You can run Istio on bare metal the same as you would run on public cloud providers.

Once installed Istio will deploy envoy proxies to all your pods as sidecar containers where Istio is enabled. To enable you could add manual injection into existing pods or automate an injection at the namespace level.

Manual injection

```istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f```

Automatic injections

```kubectl label namespace default istio-injection=enabled```

These sidecar pods act as a proxy so all inbound/outbound traffic within the pods have to traverse through these proxies. This gives you granular control over how your traffic flow within your cluster. Traffic is controlled via declarative CRD (custom resources definition files) these are essentially YAML files similar to other Kubernetes resource files.

To control inbound/outbound traffic to your entire cluster Istio implements gateways. Ingress gateway is enabled by default and allows all traffic incoming. It runs a deployment in the namespace istio-system.

```istio-ingressgateway     1/1     1            1           22d```

It will also deploy a load balancer service with a public IP, being on bare metal this IP will come from our Packet IP pool. To use ingress and define traffic apply the following CRD files.

Create a gateway which links to the ingress gateway

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
EOF
```

Create a virtual service which configures a route for entering the gateway

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```
Egress gateway is not enabled by default to enable run the following command when installing Istio

```--set values.gateways.istio-egressgateway.enabled=true```

It will also run as a deployment under the namespace istio-system.

```istio-egressgateway      1/1     1            1           18d```

To allow direct traffic through the egress gateway out to and external service you would apply ServiceEntry CRD files. The example below will allow edition.cnn.com out through the gateway as defined.

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: cnn
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
EOF 
```
---

### SDS (secret discovery service) with cert-manager

SDS automates the part of rotating keys and adding secrets. The most important benefit of SDS is to simplify certificate management. Without this feature, in k8s deployment, certificates must be created as secrets and mounted into the proxy containers. If certificates are expired, the secrets need to be updated and the proxy containers need to be re-deployed. With SDS, a central SDS server will push certificates to all envoy proxy instances. If certificates are expired, the server just pushes new certificates to Envoy instances, Envoy will use the new ones right away without re-deployment.

Benefits of using SDS

- The keys and certs can be dynamically delivered to the Envoy
-  Key and cert rotation happens on the fly
-  The private key will never leave node, private    key is stored in memory, not in files and      mounted on the pod.

You can integrate SDS with [Cert-manager](https://docs.cert-manager.io/en/latest/) and an upstream certificate issuer such as [let’s encrypt](https://letsencrypt.org/).

Istio has many different features ranging from implementing blue/green and canary deployments, but by just making use of the added envoy proxies and ingress/egress gateway you can have a fully secure service mesh cluster where you have complete control over traffic coming in and out of your cluster as well as a robust cert-manager implementation. These are key areas that are needed to make your cluster secure as possible.

---

### Wrap up!

In conclusion, there are many benefits to using bare metal to run your Kubernetes cluster most noticeably low latency, cheaper hourly rates, and fast connectivity. Along with popular open-source tools such as metalLB and Istio you can host a fully secure service mesh cluster ideal for any enterprise business requirements. Using Packet as a bare-metal cloud provider makes this task even easier by making use of their highly mature platform and their years of experience in the bare metal space. This gives your business peace of mind knowing your cluster is hosted on your own bare metal infrastructure and you are in full control!


