# Glossary

Below is a list of terminology that is good to know in order to better understand the overall application.

- Remote Direct Memory Access (RDMA) - is a technology that uses specialized [network interface cards](https://en.wikipedia.org/wiki/Network_interface_controller) to access the [RAM](https://en.wikipedia.org/wiki/Random-access_memory) of a process running in another computer. It circumvents the CPU of both systems allowing it to perform [zero-copy](https://en.wikipedia.org/wiki/Zero-copy) of data from one computer to another. RDMA allows for staggering amounts of speed when transfering data (in the 100Gb/s), without having to involve the CPU for encoding and decoding the information, making it far more efficient than using traditional mechanisms for transfering data.
- Namespace - a tool used for isolating resources from one another.
	- Network Namespace - separates different processes from being able to access the network interfaces and other network related devices from one another.
	- Filesystem Namespace - allows for isolation when it comes to file systems.
- Container - a namespace that isolates a process from other running on the system.
- Kubernets - an orchestration system for applications that exist within a container such as docker. The orchestration system allows for managing a cluster of nodes running containerized applications.
	- Pod - can be one or more containers that share the same network namespace, but have different filesystem namespaces. Must run on at least one node in the cluster.
	- Daemon Set - a pod that is run across all nodes in the cluster.
- Single Root Input/Output Virtualization (SR-IOV) - is a standard for allowing network interfaces on a system to appear in software more then there physically are. For example, an RDMA card could be virtualized (depending on what the hardward can support) to appear as 127 individual network interfaces in a software system, when in fact there only exists a single network card and interface.
	- Virtual Function (VF) - a representation in software of a physical interface.
	- Physical Function (PF) - an interface used to manage all VF's.

