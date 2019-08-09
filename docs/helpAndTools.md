# Helpful Commands and Tools
Here are some helpful commands and utilities that can assist in debugging and viewing the status of a Kubernetes system outfitted with our solution.



## Kubernetes Commands

Retrieve a list of all the nodes within a Kubernetes cluster (run this command on the master node of the cluster, or on a node where 'kubectl' is configured to access the Kubernetes API server process):
```
kubectl get nodes -o wide
```

Retrieve a list of all pods in the cluster:
```
kubectl get pods -o wide
```

Retieve a list of "Kubernetes System" pods. Pods deployed as part of the RDMA Hardware Daemon Set and Dummy Device Plugin will show up here:
```
kubectl get pods -o wide --namespace=kube-system
```

Delete a pod:
```
kubectl delete pods <POD_NAME>
```

Delete a Dameon Set:
```
kubectl delete ds --namespace kube-system <DAEMON_SET_NAME>
```

View the specific details of a pod:
```
kubectl describe pods <POD_NAME>
```

### Viewing Information on RDMA Virtual Functions for a Node

Viewing the virtual functions configured on an interface:
```
ip link show <INTERFACE_NAME>
	ex: ip link show enp4s0f0
```
Note that the version of iproute2 (the package that provides the `ip` utility) on Ubuntu that we were using has a bug that causes it to crash when displaying VFs sometimes. To avoid this, use Mellanox's version of the utility, which should already be installed if you have installed Mellanox's OFED drivers for their RDMA cards:
```
/opt/mellanox/iproute2/sbin/ip link show <INTERFACE_NAME>
```

### Executing a Shell Within a Kubernetes Container
To do this, first access the command line on the node in the cluster that the container is running on (ie: using SSH). Next, run `docker ps` to determine all of the Docker containers running on that node. Search this list for the container you are interested in (the `CREATED` time of the container, the `COMMAND` run inside of it, and the container `IMAGE` used can all be helpful here). Each pod will have at least two containers, once 'pause' container and one or more other application containers. When executing a shell, you should do so in one of the application containers. Once you have located the correct containers, find its `CONTAINER ID` and execute `docker exec -it <container_id> /bin/bash` (note that this assumes the container has the `/bin/bash` executable inside of it, adjust to your environment as necessary).



##Errors
A couple of misc. issues and how to address them.

### Kubernetes Failing to startup
If you receive an error similair to the following when running a `kubectl` command:
```
The connection to the server 129.21.34.14:6443 was refused - did you specify the right host or port?
```
Despite `kubectl` being properly configured to access the Kubernetes API server, then the Kubernetes control processes are most likely not running. To verify that this is the case, you can list the Docker containers running on the master node of the cluster:
```
docker ps
```
If this shows no containers are running on the master node, it may be an error caused by Kubernetes failing to start due to swap space being enabled (you can look this up online for more details on why this occurs). A simple solution is to disable swapping on the master node of the cluster:
```
sudo swapoff -a
```
Once you have run this command, wait a minute or so and check again for Docker containers and a `kubectl` connection.



## External Documentation
Links to other webpages and documentation that we found useful.

### Mellanox Documentation
- Guide to deploying Mellanox's solution for RDMA in Kubernetes containers (the solution that our project was inspired by/builds off of) [link](https://community.mellanox.com/s/article/kubernetes-ipoib-ethernet-rdma-sr-iov-networking-with-connectx4-connectx5)

- Configuring the maximum and minimum rate limits on an RDMA SR-IOV virtual function: [link](https://community.mellanox.com/s/article/howto-configure-rate-limit-per-vf-for-connectx-4-connectx-5)

- Configuring other attributes on a virtual function: [link](https://community.mellanox.com/s/article/howto-set-virtual-network-attributes-on-a-virtual-function--sr-iov-x)

- Sharing a single RDMA hardware interface between containers without using SR-IOV (doesn't provide the advantages of per-VF bandwidth reservation and limiting, but is useful to know about): [link](https://community.mellanox.com/s/article/kubernetes-rdma--infiniband--shared-hca-with-connectx4-connectx5)

- Mellanox RDMA driver manual for Ubuntu 16.10: [link](http://www.mellanox.com/related-docs/prod_software/Ubuntu_16_10_Inbox_Driver_User_Manual.pdf)

- Mellanox RDMA programming manual: [link](http://www.mellanox.com/related-docs/prod_software/RDMA_Aware_Programming_user_manual.pdf)

- Mellanox OFED manual (contains extensive information on what features Mellanox hardware/firmware has support for): [link](https://docs.mellanox.com/display/MLNXOFEDv461000/MLNX_OFED+Documentation+Rev+4.6-1.0.1.1)
  - Blog post that covers what OFED is and what role it plays in the system: [link](https://www.rohitzambre.com/blog/2018/2/9/for-the-rdma-novice-libfabric-libibverbs-infiniband-ofed-mofed)
  - Further information about Mellanox's OFED implementation and the Linux kernel modules that are a part of it: [link](https://community.mellanox.com/s/article/mellanox-linux-driver-modules-relationship--mlnx-ofed-x)

- Slides from a Mellanox presentation about RDMA: [link](https://events.static.linuxfound.org/sites/events/files/slides/containing_rdma_final.pdf)

### Other
- Article on what SR-IOV is and how it works: [link](https://blog.scottlowe.org/2009/12/02/what-is-sr-iov/)
- Information about the CNI standard: [link](https://github.com/containernetworking/cni)
  - Specifically, see the specification page itself: [link](https://github.com/containernetworking/cni/blob/master/SPEC.md)



## External Repositories
Links to the source code of other projects that are relevant.

### Mellanox Repositories
 - Mellanox RDMA Device Plugin: [link](https://github.com/Mellanox/k8s-rdma-sriov-dev-plugin)
 - Mellanox CNI Plugin: [link](https://github.com/Mellanox/sriov-cni)
 - Configuration files that define the contents of Mellanox RDMA Docker container images: [link](https://github.com/Mellanox/mofed_dockerfiles)

### RDMA Related Repositories:
 - "Hello World" type example for RDMA programming: [link](https://github.com/wangchenghku/rdma_handout)
   - Further information on the basics of writing an application that uses RDMA: [link](https://opensourceforu.com/2016/09/fundamentals-of-rdma-programming/)
   - RDMA 'Verbs' specification itself: [link](http://www.rdmaconsortium.org/home/draft-hilland-iwarp-verbs-v1.0-RDMAC.pdf)
 - PerfTest (RDMA toolkit that contains `ib_send_bw`, etc.): [link](https://github.com/linux-rdma/perftest/tree/master/src)

   
