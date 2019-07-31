# Components

The following are a detailed description of each of the components that make up the solution.

## RDMA Hardware Daemon Set
The RDMA Harware Daemon Set is a [Kubernetes Daemon Set](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) that has two tasks. The first task is to initialize the RDMA SRIOV enabled interfaces on a node and the second task is to create a RESTful endpoint for providing metadata about the PF's and their associated VF's that exist on a given node that are part of the Kubernetes cluster. Further detail about each of the containers within the RDMA Hardware Daemon set can be seen below:

1. Init Container - the init container is a privileged container that runs as a [Kubernetes Init Container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/), in simplified terms this means it is run before any other container within the pod. The reason that the container needs to be run as a privileged container is because it will be working on a nodes network devices, which means it needs special access in order to configure them. The init container does two things, the first is to scan all the available interfaces on the node and determine if it is an RDMA device and whether SRIOV is enabled. For all interfaces that meet those two requirements, the init container configures each interfaces VF's to be available and ready to run.
2. Server Container - the server container is an unprivileged container that scans through all available interfaces when starting up. It then makes a list of all interfaces that are SRIOV enabled and upon a RESTful get request from a user will return associated metadata about the container. The server container only scans the interfaces at startup because SRIOV devices can only change configurations upon a machine restart, which would rerun the init container and then the server container. The RESTful endpoint serves data in a JSON formatted list of PF's and each PF has an internal list of VF's, more info can be found [here](https://github.com/rit-k8s-rdma/rit-k8s-rdma-ds). The RESTful endpoint that the server container sets up is bound to the host network.

Both the init container and the server container run under the same RDMA Hardware Daemon Set pod. When configuring how to install this pod please look at [daemon set install instructions](install.md#daemon_set_install_instructions) for more detail.


## Scheduler Extender

## CNI

## Dummy Device Plugin
The Dummy Device Plugin is a stop gap measure for the current system. The directory `/dev/infiniband` is needed within any pod that requires RDMA. In order to access devices in this directory a container either needs to be run in privileged mode or a [Device Plugin](https://kubernetes.io/docas/concepts/extend-kubernetes/compute-storage-net/device-plugins/) can return a directory that a container will have privileged access to. For obvious reasons we opted to create a Device Plugin for the sole purpose that it will give **any container** that requests an RDMA device the proper access to `/dev/infiniband`. This Dummy Device Plugin does not act like any other Device Plugin where it is meant to manage a specialized resources; that is done by the RDMA Hardware Daemon Set, Scheduler Extender, and CNI components. When configuring how to install this pod please look at [dummy device plugin installation](install.md#dummy_device_plugin_installation) for more detail.


