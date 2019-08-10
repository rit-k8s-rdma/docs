# Glossary

Below is a list of terminology that is good to know in order to better understand the overall application.

## Remote Direct Memory Access (RDMA)
A technology that uses specialized [network interface cards](https://en.wikipedia.org/wiki/Network_interface_controller) to access the [RAM](https://en.wikipedia.org/wiki/Random-access_memory) of a process running in another computer. The RDMA protocol allows data transfers to bypass the CPU and operating system kernel of both the sending and receiving computers. This enables RDMA interfaces to provide large amounts of bandwidth (40-100+ Gbps per interface), making it useful for bandwidth-intensive applications.

## Namespace
An entity used to manage a processes access to a resource. Processes can only access network, filesystem, IPC, etc. resources if they are running in the same network, filesystem, IPC, etc. namespace that those resources have been placed in, respectively. Containers provide the isolation layer that underlies all containerization within Linux. See [this](https://www.ianlewis.org/en/what-are-kubernetes-pods-anyway) and [this](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/) for more details.

### Network Namespace
Isolates network resources from processes. Every process running on a Linux system runs within some network namespace, though most are running in the "default" network namespace for the system. When a network interface is placed into a network namespace, only processes within that same namespace can access it. In Kubernetes, every container running within the same pod runs within the same network namespace (that is, each Kubernetes pod has its own network namespace). See [here](https://www.ianlewis.org/en/almighty-pause-container) for more information.

### Filesystem Namespace
Isolates filesystem resources from processes. Every process running on a Linux system runs within some network namespace. For conventional applications, this is often the "default" filesystem namespace with all of the systems files on it. Applications in a container typically run in their own filesystem namespace, which contains only the files packaged within that container. In Kubernetes, each container runs within its own filesystem namespace. Containers within the same pod do NOT share a filesystem namespace.

## Container
Containers are simply collections of namespaces (typically one of each type) that define the environment one or more processes will run in. Which namespaces are used for the container and what resources are present within those namespaces determine what the processes running inside that container will have access to.

## Kubernetes
An [orchestration system](https://kubernetes.io/) for applications that exist within a container such as [Docker](https://www.docker.com/). The orchestration system allows for managing containers across cluster of nodes.

### Kubelet Process
A process that runs on each worker node within a Kubernetes cluster. Kubelet is a part of Kubernetes, and is responsible for creating new pods and container, as well as reporting resource usage information back to the Kubernetes control processes.

### kube-scheduler (AKA "Kubernetes core scheduler")
One of the Kubernetes control processes. Responsible for determining which node in a Kubernetes cluster new pods will be run on. Scheduler extensions add functionality to kube-scheduler, and are registered within its configuration file.

## Pod
The smallest unit of deployment within Kubernetes. Each pod contains one or more containers, and all the containers in a pod run on the same node in a Kubernetes cluster at the same time.

## Daemon Set
A specification for a pod that is intended to be run on every worker node within a Kubernetes cluster. Deploying a Daemon Set causes every node in the cluster to run a copy of the same pod.

## Container Network Interface (CNI)
A specification that describes the way network access is provided to containerized applications. This specification can be found [here](https://github.com/containernetworking/cni)

### CNI Plugin (AKA Network Plugin)
A Kubernetes-specific type of software component that is executed on pods as they are set up and torn down. CNI plugins perform the task of attaching a network interface to the pod they are run on.

## Single Root Input/Output Virtualization (SR-IOV)
A standard for allowing physical PCI devices (referred to as PFs or "physical functions") to appear in software as many virtual devices (referred to as VFs or "virtual functions"). In this project, SR-IOV is used to split up each physical RDMA network interface into many RDMA virtual functions (allowing us to attach each virtual function to only one pod).

### Virtual Function (VF)
A hardware/firmware-backed representation of a virtual PCI device.

### Physical Function (PF)
An actual PCI device (an RDMA network interface in our case) that is split up into VFs.

