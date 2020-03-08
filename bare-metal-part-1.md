##Kubernetes running on the bare metal public cloud platform Packet!

###Part 1
Kubernetes get more widely adopted by the large enterprise market as becoming the tool to deploy their apps within a cloud-native environment. Many companies choose to host their Kubernetes cluster on bare metal infrastructure compared with running on a public cloud provider such as AWS. Both have their benefits but in this post, I will discuss the positives on how bare metal is the ideal choice for large enterprises who want to start using cloud-native technology and want to reap to rewards of bare metal such as speed, security and full access to the underlying infrastructure.

When it comes to bare metal consumers are often worried can they still use all the available open-source tools in the cloud-native ecosystem? the majority of the time the answer is there is no difference in the way you install tools on your cluster via either bare metal instances or public cloud virtual machines. I will go over this more in part 2 of this post.

Bare metal infrastructure essentially means you have full access to the underlying machine, there is no hypervisor installed. You could buy servers in a data center and let their operation teams rack, stack and host those servers, then you would install Kubernetes. Now, this would be a bare-metal cluster but this is a legacy on-prem way of achieving. There is also a lot of added hassle with this such as maintaining the physical hardware, arranging licenses for operating systems and choosing prime physical rack space in already overcrowded data center colocations.

The most optimal way of running a bare-metal cluster is to use a bare-metal public cloud provider this blends benefits of pure public cloud provider such as strong user interface, on-demand machines, and region-wide availability with all the benefits of bare metal such as full access to infrastructure and low latency speeds between servers.
The pioneer of the bare metal public cloud platform are a company called Packet.
I recently got a chance to trail there platform and deploy a Kubernetes cluster onto their super speedy powerful on-demand instances. I will discuss further in this post on all the nice features and benefits that come with using the great service Packet has built.

---

###What is Packet?
[Packet](https://www.packet.com/) is a leading bare metal public cloud provider with over 22+ locations worldwide, so you never far from deploying bare metal in your chosen region. It is fully integrated with the most well known DevOps tool such as Terraform and Ansible. It has an easy to use simplistic user interface in which you can deploy bare metal in a few clicks, it's very fast to deploy averaging in under 60 seconds. There are some nice added features exclusive to Packet once instances are deployed you have SOS out of band access. In case you lose access to your instance due to misconfiguration or bad network connectivity, there is a peace of mind you will never be fully locked out from your machine. This can be crucial in troubleshooting situations.
There is also an option to bring your own SSH keys to connect to your instances which is usually not an option with most public cloud providers. This gives you flexibility and also can reduce your security footprint by not having numerous keys lying around.
Out of the box within the UI, there is also a timeline log so you can monitor who access and made changes to your instances as well time and date. This is always a nice added feature to have which requires no added config from the user's end.
Packet offers a wide range of instance sizes from tiny to large GPU intensive, this fall in line with what most public cloud providers offer with added extra of being able to choose arm arch for CPU along with x86 offerings. In addition to this, you can expand your specification in many different areas to have the ability to have 2x 10gbps network connections or powerful Nvidia graphics, in turn, this gives you the ability to have very powerful clusters to meet the requirements of your workloads.
Packet also supports many open source projects and is linked to many key foundations such as [CNCF](https://www.cncf.io/). This show Packet is supporting cloud-native technologies and strongly embedded in the community another great reason to choose them to host your Kubernetes cluster.

---

###Latency comparison to public cloud
I recently conducted a test to see the average latency between two worker nodes in a Kubernetes cluster. The test was conducted using an iperf networking tool modified for Kubernetes usage. When the script is run it will deploy two pods one on each separate worker node and measure the latency between both pods.
Our Packet bare metal cluster has an instance size of m1.xlarge.x86 you can view the specs [here](https://www.packet.com/cloud/servers/m1-xlarge/). 
After running scripts that deployed the pods and runs the latency test results are below.

![Image of packetlate](/images/bare-metal/packetlate.png)

For AWS and instance size of m5d.16xlarge was used which is the closer spec I could find to the Packet instance, you can view specs [here](https://aws.amazon.com/ec2/instance-types/).

![Image of awslate](/images/bare-metal/awslate.png)

As you can see from result latency is almost 3x as much between pods on AWS VM instances. If you are looking for low latency within your cluster Packet is clearly the preferred option. Few things to note we also have workload running on our Packet test cluster but latency is still a lot less!

---


###Write Speed on disk comparison to public cloud
To measure disk speed on the cluster we can use the Linux dd command. We run the following command. Both cluster nodes are running SSD hard drives.

```dd if=/dev/zero of=/tmp/test1.img bs=1G count=1 oflag=dsync```

![Image of packerclusternode](/images/bare-metal/packetclusternode.png)

As you can see Packet is faster finishing at 4.3 seconds and transferred 246 MB/s.

![Image of awsclusternode](/images/bare-metal/awsclusternode.png)

---

###Cost comparison to public cloud
For testing, we were using a four-node cluster. Instances on Packet are charged at an hourly rate for on-demand instances.

```Master node 1x c2.large.arm at $1.00 per hour```
```Worker node 3x m1.xlarge.x86 at $1.70 per hour```

```To keep the cluster running for 24 hours it would cost a total of $146.40```

To run the same cluster on AWS with similar spec instances.
```Master node 1x m5d.8xlarge at $2.096 per Hour```

```Worker node 3x m5d.16xlarge at $4.192 per hour```
```To keep the cluster running for 24 hours it would cost a total of $352.129```

Although it’s hard to get the specs of each machine exactly the same, as a rough estimation we can see you can already save nearly 50% using Packet and running your workloads on bare metal.

Again this is a basic estimation price that can vary depending on instances type, region deployed and how long cluster is operational.

---

In part two I will dive into explaining how we can run a fully secure service mesh cluster on Packet using Istio and metal LB (load balancing) specifically designed for bare-metal infrastructure.
