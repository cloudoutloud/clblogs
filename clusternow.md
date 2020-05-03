## I need a cluster now!

I often find myself in need of a local Kubernetes cluster to spin up. This could be for testing, demo or general studying of latest features offered in each Kube release version.  Ideally, I want the offering to be free, lightweight with a low learning curve and of course, the most important aspect is the cluster quick and easy to set up!

### Managed in the cloud.

Public cloud offerings have matured a lot in recent years. All major providers have released their versions of a managed Kubernetes cluster offering AKS, EKS and GKE. These can be a good solid, hasslefree way of spinning up a cluster instantly. Managed offerings are a great way for enterprises to get there workloads running in the cloud in a stress-free manner. They do have some drawbacks when it comes to developers wanting something quick and easy to access.

First drawback, the most obvious one which probably springs to mind is cost. These offerings are not free and can get very expensive, especially if you leave the cluster running for days or weeks and have completely forgotten about it.

The second drawback is these managed offerings try and completely abstract the end-user from the master control plane components, this is great from a support perspective that is what you are paying money for to have ease of mind your control plane is HA and always online. 
Although this can be bad if you’re using to cluster for studying for the CKA/CKAD certification exams as you will not get exposure to the key components and will not have a core understanding of how the cluster is essentially working.

The third drawback is that the cloud vendors will often restrict what Kubernetes version you can use within your cluster. Often time the latest version released will take time to get promoted as stable within the cloud vendor as they will need to test version too. This can be bad if you need to test the latest version quickly.

The last drawback is often it’s hard to enable features on the API-server or controller-manager.
This is normally done through ‘--feature-gates’ option as explained in the Kubernetes documentation [link]. As you do not have any control over how the control plane is creating you can often be stuck looking at how you can implement these features into a managed service. Again this can be an issue for testing new features offered in each respective version release.


Now let’s have a look at some of the options we have available today in the open-source ecosystem. We will also look at the pros and cons of these options and what is truly the easiest and hasslefree way. 


### Kubeadm on Virtual box virtual machines.

For the underlying node infrastructure, we can use the popular Oracle VM virtual box this is probably the easiest and popular way to set up VM’s locally on your laptop. It is a mature product and free to use. Endless operating systems are supported and we can automate the creation of nodes with tools such as Vagrant.
Once we have our blank VM nodes provisioned we can use a tool called Kubeadm to set up the control plan. kubeadm will take care of initializing the control plane components and provide an easy option to bootstrap worker nodes into the cluster, usually via one single join command. In-turn we have a fully working cluster set up in almost half the time if we did from scratch. 

#### Pros
kubeadm easy to use and is backed by a strong community of maintainers.
kubeadm is supported on most Linux distributions.
Kubeadm is a certified Kubernetes tool 

#### Cons
Running VM locally can place strain on your laptop as VM’s within the cluster can get resource dependant.
VM OS image ios file can be large in size and take up laptop hard drive space.
Running kubeadm configuration for the first time on a new cluster, issues can arise although there is a great troubleshooting< link > section in the documentation.


### Kind

Kind is a great tool it was primarily designed for testing Kubernetes but can be used for local development and continuous integration. Essentially it uses docker containers as nodes which makes it a fast lightweight option to provision a local cluster. You will need golang and docker installed locally prior to using. Installation is as easy as running `kind create cluster` under the hood kind is using kubeadm which we discussed earlier, although this is all fully automated.

#### Pros
Quick lightweight cluster provisioned in seconds!
No strain on your local laptop, nodes are lightweight Docker containers.
Can configure to pull from local or private image registries.

#### Cons
Tool is still not in beta.
Documentation still under development.
Once the cluster is stopped it cannot be restarted, not meant to be persistent.

### Minikube 

Is a quick way to install a one-node cluster locally. Provided to be a quick and easy tool making it easy to learn and develop within a Kubernetes environment. All you need to begin with is docker installed and a suitable virtual machine. 

#### Pros
Quick and easy to set up.
Minikube is a certified Kubernetes tool.
Can be deployed as VM, container or on bare metal.
Can be installed on Linux, macOS or Windows.

#### Cons
Only a one-node cluster, can’t have a multi-node cluster environment.
Still using a virtual machine for an underlying node can take local laptop resources.
Does not support all Kubernetes features.

### Summary

In conclusion, I would say the best way to get a local lightweight cluster up and running is using kind. Most notably there is no need to use a VM this can wreak havoc on your local laptop resources and just adds another resource to maintain and keep the operating system updated.
Please see links below for official documents.

### Links

[kubeadm offical docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
[Minikube offical docs](https://kubernetes.io/docs/tasks/tools/install-minikube/)
[Virtual box](https://www.virtualbox.org/)
[Kind](https://kind.sigs.k8s.io/) 


