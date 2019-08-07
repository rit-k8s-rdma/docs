# Introduction

The main focus of this project was to enable bandwidth limiting and allow for multiple interfaces to be specified for RDMA interfaces on a per pod basis. The current Mellanox Solution has a number of pitfalls such as an inaccurate state between the CNI and the Device Plugin. The Device Plugin is managed by Kubernetes, so Kubernets decides which interface to allocate to a given container. This is not reflected accurately in the CNI which may give it a different interface. In the current solution a pod may have 3 containers that each request a single RDMA interface; in the solution the Device Plugin removes 3 RDMA interfaces from the available pool of interfaces, but the CNI only allocates a single interface; the state of the CNI and the Device Plugin are incorrect. One of the largest problems is that Mellanox treats RDMA interfaces as a container specified resource, when in reality network namespaces are shared across pods, so any container in a pod has access to all of the same interfaces. This becomes a problem because when specifying Device Resources within a pod yaml, they are on a per container basis. For all of the reasons above the Device Plugin approach was abandoned because it could not accommodate our goals and instead we opted for a new architecture.

## Architecture

The main system architecture for our design can be seen below; the green color specifies our components for our design and the yellow color specify Kubernetes components:

![Screenshot](assets/architecture.png)

The main workflow for how a pod would deployed in our system begins with a request for deploying a pod going to the master nodes Kubernetes Control Process. After that request is seen the Kubernetes Scheduler creates a list of possible nodes that the pod can be placed on based on requirements of the pods yaml. This list then gets sent to our [Scheduler Extension](components.md#scheduler_extension), which is in charge of deciding which nodes can support the RDMA requirements specified in the pods yaml.

The [Scheduler Extension](components.md#scheduler_extension) contacts each nodes [RDMA Hardware Daemon Set](components.md#rdma_hardware_daemon_set), which returns a JSON formatted list of information about a nodes RDMA VF's back to the [Scheduler Extension](components.md#scheduler_extension). The [Scheduler Extension](components.md#scheduler_extension) than processes all of the information to find a valid node that can meet the minimum bandwidth requirements of each of the requested interfaces that is specified in the pods yaml; the list of nodes whether blank or empty is sent back to the Kubernetes Core Scheduler. If no node is valid after the scheduling calls an error is raised and the pod is not placed, this error can be seen with a Kubernetes describe command of why the pod was not placed.

Assuming that the pod was able to placed on at least one valid node, the Kubelet process on the valid node gets called to setup the pod. During the pods setup process the (CNI)[components.md#cni] is called to setup the network of the pod. The (CNI)[components.md#cni] first contacts the [RDMA Hardware Daemon Set](components.md#rdma_hardware_daemon_set) on the node that is running to get an up to date list of the state of the node. It then runs the same algorithm that the [Scheduler Extension](components.md#scheduler_extension) had run to find the correct placements of interfaces to meet the requirements of the bandwidth limitations. The (CNI)[components.md#cni] is atomic operation, so it either completes the setup of the pod or fails and rollbacks all changes made to any interfaces. Once the (CNI)[components.md#cni] finishes, the response is sent back to the Kubelet process of the node.

## Limitations

There are a couple limitations when it comes to our solution:
- Mellanox Vendor - the following has only been test on a Mellanox Card
- Data Plane Development Kit (DPDK) - the Mellanox solution may work with DPDK, it has not been tested with our solution (changes to (CNI)[https://github.com/rit-k8s-rdma/rit-k8s-rdma-sriov-cni] required)
- Shared RDMA Device - the Mellanox solution may work with a shared RDMA interface, it has not been tested with our solution (changes to (CNI)[https://github.com/rit-k8s-rdma/rit-k8s-rdma-sriov-cni] required)
- Dummy Device Plugin - the current solution requires the use of Device Plugin to give access to `/dev/infiniband` for open issues in Kubernets that can be found [here](https://github.com/kubernetes/kubernetes/issues/5607) and [here](https://github.com/kubernetes/kubernetes/issues/60748). The main problem is Kubernetes does not have the ideas of a `device` directory that [Docker has `--device`](https://docs.docker.com/engine/reference/commandline/run/).

## Future Work
- More Vendors - making the solution more interface driven so it can be adapted to more vendors then just Mellanox.
- Migrating CNI - the CNI is currently at an older version and we had to bootstrap the newer one, it should be upgraded.
- Scheduling - opening up the scheduler to be more adaptable to customizable scheduling algorithms.

#### Our Approach in Short
Problems Addressed:
 - VF's are specified per container
 - Network plugin only ever gives one vf per pod, this means that the device plugin which tracks the amount of available VF's does not maintain an accurate count of VF's being used
 - The VF's selected by Kuberentes to be allocated, may not match those actually allocated by the network plugin

Solution:
 - Device Plugin (RDMA)
   - Changes to a DameonSet 
   - Stores the current state of the nodes VF resources
   - Can be quierried through an API
 - Schedular extender
   - Queries each DameonSet on each node and then filters the possible nodes that a pod can be deployed on based on resource requirements in the annotations for the pod
 - Network Plugin (SRIOV)
   - Modify it to handle read pods meta-data
     - Read the amount of VF's
     - Read the bandwidth limitation on each VF
   - Modify plugin to set the bandwidth limits from read metadata
   - Add ability to set `min_tx_rate` and `max_tx_rate`
      
