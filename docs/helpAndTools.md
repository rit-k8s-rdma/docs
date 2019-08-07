# Helpful Commands and Tools
Here are some helpful commands and some repositories of where to find information that we found helpful.

## Kubernetes Commands

Get nodes:
```
kubectl get nodes
```

Get pods:
```
kubectl get pods -o wide
```

Get pods in namespace:
```
kubectl get pods -o wide --namespace=kube-system
```

Delete all pods, services, and anything else:
```
kubectl delete daemonsets,replicasets,services,deployments,pods,rc --all
```

To reset the master node
*NOTE* must reset up all of the clients for kubernetes:
```
sudo kubeadm reset
```

Delete config map if already exists:
```
kubectl delete configmap <config-map-name> -n kube-system
```

Delete DameonSet Extension (the DaemeonSet extension is the pod that runs the Mellanox RDMA plugin)
```
kubectl delete ds --namespace kube-system <dameon-set-name>
```

Deleting a kubernetes pod
```
kubectl delete pod <pod-name>
```
<pod-name> is the name that shows up when you do "kubectl get pods"

### Interface on RDMA Nodes
Assign IP to a specific interface:
```
sudo ifconfig  <interface-name> <ip>
```

Bring interface up:
```
sudo ifconfig <interface-name> up
```

Getting VF information:
```
/opt/mellanox/iproute2/sbin/ip link show <interface>
```

#### Changing the number VFs

To change the VFs on RDMA node first run the following command in order to configure the Mellanox NIC:
```
sudo mst start
```

To enable SRIOV and change the number of VFs type in:
```
sudo mlxconfig -d /dev/mst/<mellanox-switch> set SRIOV_EN=1 NUM_OF_VFS=120
```

Then restart the machine with a friendly message:
```
sudo shutdown -r now 'Updating Mellanox Config'
```

#### Finding info about Mellanox NIC
Start the Mellanox device:
```
sudo mst start
```

List information about the driver:
```
sudo mlxconfig -d /dev/mst/mt4119_pciconf0 q
```

Find the number of Virtual Functions that were created and make sure the SRIOV environment has been enabled:
```
sudo mlxconfig -d /dev/mst/mt4119_pciconf0 q | grep "NUM_OF_VFS"
sudo mlxconfig -d /dev/mst/mt4119_pciconf0 q | grep "SRIOV_EN"
```

### Testing Containers on Nodes or Containers
If you want to run this on a container, go to the node that the pod is running on and run `docker ps`, find the Container ID of the pod you launched (NOT the pause container) and run `docker exec -it <container_id> /bin/bash`. After that complete the instructions below.

After you are inside a container or on a system that has a mellanox card running the the following:
```
ifconfig -a
```
This will list all the interfaces available.

Then run:
```
ibdev2netdev
```
This will give you a list of adapters that you will need in order to connect them to interfaces on the actual system.

One container will be the server and the other will be the client:
 - Server: `ib_send_bw -d <rdma_adapter_name> -i 1 -F --report_gbits --run_infinitely`
   - <rdma_adapter_name> is taken from running "ibdev2netdev -v"
     - ex: mlx5_2
   - Command Ex: `ib_send_bw -d mlx5_2 -i 1 -F --report_gbits --run_infinitely`
 - Client: `ib_send_bw -d <rdma_adapter_name> -i 1 -F --report_gbits <server_ip> --run_infinitely`
   - <server_ip> is the IP of the pod that the 'server' testing command was run on
   - Command Ex: `ib_send_bw -d mlx5_5 -i 1 -F --report_gbits 10.55.206.84 --run_infinitely`


##Errors
Common errors

### Kubernetes Failing to startup
If you receive something similar to the following error:
```
The connection to the server 129.21.34.14:6443 was refused - did you specify the right host or port?
```
Most likely it is because the process `kubelet` failed to start. For some reason it requires the swap space to be off.
Run the following command to turn it up and after it the kubelet process should start running.
```
swapoff -a
```
### Unable to Open File Descriptor
This has something to do with how the VFs are allocated and changed (AKA we are not entirely sure, but you should follow this guide or it will fail to latch to sockets). Here is the example error:
```
Couldn't connect to 10.55.206.82:18515
Unable to open file descriptor for socket connection Unable to init the socket connection
```
To remdy this error as well as correctly change number of VFs do the following.

On Skya (the kubelet master) run:
```
kubectl delete pod <pods>
```

## Repositories and Guides
Lots of information of repos and information that is helpful:

### Mellanox Rate Limiting
The commands are taken from https://community.mellanox.com/s/article/kubernetes-ipoib-ethernet-rdma-sr-iov-networking-with-connectx4-connectx5
*NOTE* throughout the notes mt4119_pciconf0 is used, in reality run `ls /dev/mst/<name>` to find the name of your device

- "how to configure rate limit per VF": [https://community.mellanox.com/s/article/howto-configure-rate-limit-per-vf-for-connectx-4-connectx-5](https://community.mellanox.com/s/article/howto-configure-rate-limit-per-vf-for-connectx-4-connectx-5)
- "how to set virtual network attributes on a VF": [https://community.mellanox.com/s/article/howto-set-virtual-network-attributes-on-a-virtual-function--sr-iov-x](https://community.mellanox.com/s/article/howto-set-virtual-network-attributes-on-a-virtual-function--sr-iov-x)

### Repos
#### Mellanox
 - Physical: [https://github.com/Mellanox/k8s-rdma-sriov-dev-plugin](https://github.com/Mellanox/k8s-rdma-sriov-dev-plugin)
 - HCA-only: [https://community.mellanox.com/s/article/kubernetes-rdma--infiniband--shared-hca-with-connectx4-connectx5](https://community.mellanox.com/s/article/kubernetes-rdma--infiniband--shared-hca-with-connectx4-connectx5)
 - SR-IOV: [https://github.com/Mellanox/sriov-cni](https://github.com/Mellanox/sriov-cni)
    - Info: [https://blog.scottlowe.org/2009/12/02/what-is-sr-iov/](https://blog.scottlowe.org/2009/12/02/what-is-sr-iov/)
 - DockerFiles: [https://github.com/Mellanox/mofed_dockerfiles](https://github.com/Mellanox/mofed_dockerfiles)

#### Container Networking Interface
 - [https://github.com/containernetworking/cni](https://github.com/containernetworking/cni)
 - Specification:
    - [https://github.com/containernetworking/cni/blob/master/SPEC.md](https://github.com/containernetworking/cni/blob/master/SPEC.md)

#### Linux Kernel:
 - [https://github.com/torvalds/linux](https://github.com/torvalds/linux)
    - [https://github.com/torvalds/linux/tree/master/drivers/infiniband/core](https://github.com/torvalds/linux/tree/master/drivers/infiniband/core)
    - [https://github.com/torvalds/linux/tree/master/drivers/net/ethernet/mellanox/mlx5/core](https://github.com/torvalds/linux/tree/master/drivers/net/ethernet/mellanox/mlx5/core)

### RDMA Related Information:
 - "Hello World" type example:
    - [https://github.com/wangchenghku/rdma_handout](https://github.com/wangchenghku/rdma_handout)
 - PerfTest (ib_send_bw, etc.):
    - [https://github.com/linux-rdma/perftest/tree/master/src](https://github.com/linux-rdma/perftest/tree/master/src)
 - Information on the basics of writing an application that uses RDMA:
    - [https://opensourceforu.com/2016/09/fundamentals-of-rdma-programming/](https://opensourceforu.com/2016/09/fundamentals-of-rdma-programming/)
 - RDMA Verbs specification:
    - [http://www.rdmaconsortium.org/home/draft-hilland-iwarp-verbs-v1.0-RDMAC.pdf](http://www.rdmaconsortium.org/home/draft-hilland-iwarp-verbs-v1.0-RDMAC.pdf)
 - "RDMA Core":
    - Mellanox:
        - [https://github.com/Mellanox/rdma-core](https://github.com/Mellanox/rdma-core)
    - "linux-rdma":
        - [https://github.com/linux-rdma/rdma-core](https://github.com/linux-rdma/rdma-core)
 - Manuals:
    - Ubuntu Driver Install: [http://www.mellanox.com/related-docs/prod_software/Ubuntu_16_10_Inbox_Driver_User_Manual.pdf](http://www.mellanox.com/related-docs/prod_software/Ubuntu_16_10_Inbox_Driver_User_Manual.pdf)
    - RDMA Programming Manuals: [http://www.mellanox.com/related-docs/prod_software/RDMA_Aware_Programming_user_manual.pdf](http://www.mellanox.com/related-docs/prod_software/RDMA_Aware_Programming_user_manual.pdf)
    - MLNX OFED Manual: [https://docs.mellanox.com/display/MLNXOFEDv451010/MLNX_OFED+v4.5-1.0.1.0+Documentation](https://docs.mellanox.com/display/MLNXOFEDv451010/MLNX_OFED+v4.5-1.0.1.0+Documentation)
 - Info: [http://www.rdmamojo.com/2014/03/31/remote-direct-memory-access-rdma/](http://www.rdmamojo.com/2014/03/31/remote-direct-memory-access-rdma/)
 - Other:
    - HustCat CNI RDMA device Plugin: [https://docs.google.com/document/d/1PPOYOdTstnG8-XEHgoCkHEOw5VUOt1blAQNu3MnXuY8/edit](https://docs.google.com/document/d/1PPOYOdTstnG8-XEHgoCkHEOw5VUOt1blAQNu3MnXuY8/edit)
    - Mellanox Presentation about RDMA: [https://events.static.linuxfound.org/sites/events/files/slides/containing_rdma_final.pdf](https://events.static.linuxfound.org/sites/events/files/slides/containing_rdma_final.pdf)
    - Mellanox Qos Description: Traffic Classes: [https://community.mellanox.com/s/article/network-considerations-for-global-pause--pfc-and-qos-with-mellanox-switches-and-adapters](https://community.mellanox.com/s/article/network-considerations-for-global-pause--pfc-and-qos-with-mellanox-switches-and-adapters)
    - Mellanox Recommended Configuration for deployment: [https://community.mellanox.com/s/article/recommended-network-configuration-examples-for-roce-deployment](https://community.mellanox.com/s/article/recommended-network-configuration-examples-for-roce-deployment)
    - MOFED and OFED Explanation: [https://www.rohitzambre.com/blog/2018/2/9/for-the-rdma-novice-libfabric-libibverbs-infiniband-ofed-mofed](https://www.rohitzambre.com/blog/2018/2/9/for-the-rdma-novice-libfabric-libibverbs-infiniband-ofed-mofed)
    - Mellanox QoS Infiniband Deployment: http://www.mellanox.com/pdf/whitepapers/deploying_qos_wp_10_19_2005.pdf
    - Description of what an HCA is: [https://www.ibm.com/support/knowledgecenter/TI0003M/p8ha1/smhostchanneladapter.htm](https://www.ibm.com/support/knowledgecenter/TI0003M/p8ha1/smhostchanneladapter.htm)
    - Mellanox Driver Description: [https://community.mellanox.com/s/article/mellanox-linux-driver-modules-relationship--mlnx-ofed-x](https://community.mellanox.com/s/article/mellanox-linux-driver-modules-relationship--mlnx-ofed-x)
    - Mellanox RDMA-SRIOV Setup: [https://community.mellanox.com/s/article/kubernetes-ipoib-ethernet-rdma-sr-iov-networking-with-connectx4-connectx5](https://community.mellanox.com/s/article/kubernetes-ipoib-ethernet-rdma-sr-iov-networking-with-connectx4-connectx5)
    - Mellanox RDMA-HCA Setup: [https://community.mellanox.com/s/article/kubernetes-rdma--infiniband--shared-hca-with-connectx4-connectx5](https://community.mellanox.com/s/article/kubernetes-rdma--infiniband--shared-hca-with-connectx4-connectx5)
   
