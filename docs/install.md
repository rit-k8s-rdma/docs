# Install

## Prerequisites

Before installing this system, you should have a working Kubernetes cluster set up. The following prerequisites must be met:
 - Kubernetes version 1.13
 - Golang version 1.12
   - Version 1.12 or greater is necessary to compile the software components of the system.
 - Mellanox OFED version 4.6-1.0.1.1 or greater
   - The firmware on Mellanox RDMA cards should be updated to the latest available version.

## Hardware Setup
  This section covers the configuration of Mellanox RDMA hardware in preparation for using SR-IOV.

1. Enable SR-IOV in the BIOS of an machine with RDMA hardware installed.
2. Load the Mellanox driver modules for making configuration changes to the RDMA hardware.

	Run:

        sudo mst start

	You should expect to see output similar to the following:
     
        Starting MST (Mellanox Software Tools) driver set
        Loading MST PCI module - Success
        Loading MST PCI configuration module - Success
        Create devices
        Unloading MST PCI module (unused) - Success

3. Determine the path to the PCI device for the RDMA hardware card.

	Run:

        sudo mst status

	You should expect to see output similar to the following:

        MST modules:
        ------------
            MST PCI module is not loaded
            MST PCI configuration module loaded

        MST devices:
        ------------
        /dev/mst/mt4119_pciconf0         - PCI configuration cycles access.
                                           domain:bus:dev.fn=0000:04:00.0 addr.reg=88 data.reg=92
                                           Chip revision is: 00

	The /dev/mst/ path is the path to the device:

        /dev/mst/mt4119_pciconf0

4. Query the status of the device to determine whether SR-IOV is enabled, and how many virtual functions are configured.

	Run:

        mlxconfig -d <pci_device_path> q

	Here, <pci_device_path\> is the path to the PCI device determined in the previous step. Ex:

        /dev/mst/mt4119_pciconf0

	You should expect to see output similar to the following:

        Device #1:
        ----------

        Device type:    ConnectX5       
        Name:           MCX556A-ECA_Ax  
        Description:    ConnectX-5 VPI adapter card; EDR IB (100Gb/s) and 100GbE; dual-port QSFP28; PCIe3.0 x16; tall bracket; ROHS R6
        Device:         /dev/mst/mt4119_pciconf0

        Configurations:                              Next Boot
                 MEMIC_BAR_SIZE                      0               
                 MEMIC_SIZE_LIMIT                    _256KB(1)       
                 HOST_CHAINING_MODE                  DISABLED(0)
                 ...
                 NUM_OF_VFS                          120
                 ...
                 SRIOV_EN                            True(1)
                 ...

	The lines of interest to us are:

        SRIOV_EN                            True(1)

	Which indicates whether or not SR-IOV has been enabled on the RDMA card.

	And:

        NUM_OF_VFS                          120

	Which indicates how many SR-IOV virtual functions have been configured on the card.

	We want to ensure that SR-IOV is enabled, and the number of virtual functions is configured to the largest amount the card will support.

5. Enable SR-IOV and confuigure number of VFs.

	Run:

        mlxconfig -d <pci_device_path> set SRIOV_EN=1 NUM_OF_VFS=<max_num_vfs>

	Here, <pci_device_path\> is the path determined in step 3, and <max_num_vfs\> is the highest number of virtual functions that the RDMA hardware card supports. This can be found in the documentation for that card (this can typically be found in the firmware manual).

	For example:

        mlxconfig -d /dev/mst/mt4119_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=120

	Choose 'yes' when asked whether to apply the configuration.

6. Reboot the machine.

7. Verify that the modification worked correctly.

	Run:

        sudo mst start
        sudo mst status
        mlxconfig -d <pci_device_path> q | egrep 'NUM_OF_VFS|SRIOV_EN'

	Ensure that the output of the last command matches the changes you have made prior to rebooting.





## RDMA Hardware Daemon Set

This section covers the installation of the RDMA Hardware Daemon Set onto all of the worker nodes in the Kubernetes cluster.

1. Use Kubernetes to deploy the Daemon Set to all the nodes in the cluster.

	On the master node of the Kubernetes cluster, run:

        kubectl apply -f <rdma_daemonset_yaml>

	Where <rdma_daemonset_yaml\> is a YAML file that specifies the details of the Daemon Set. This file can be found at:

        https://github.com/rit-k8s-rdma/rit-k8s-rdma-ds/blob/master/rdma-ds.yaml

	Applying this Daemon Set configuration on the Kubernetes cluster will cause each worker node to pull container images built as part of this project from our [Docker Hub repository](https://hub.docker.com/u/ritk8srdma).

If you would like to build this Docker image yourself, the instructions are available within the [RDMA Hardware Daemon Set repository](https://github.com/rit-k8s-rdma/rit-k8s-rdma-ds).

2. Verify that the RDMA Hardware Daemon Set is running on each worker node in the cluster.

	On the master node of the Kubernetes cluster, run:

        kubectl get pods -o wide --namespace kube-system

	Within the output for this command, you should see several lines with the name: `rdma-ds-*` (one for each worker node in the cluster). The `status` column of each of these pods will show `Init:0/1` while the pod is performing hardware enumeration and initialization of the SR-IOV enabled RDMA hardware. One this has completed (it may take several minutes if there are a large number of virtual functions configured on a host), the status of the pods should switch to `Running`.

## Scheduler Extension

This section covers the installation of the Scheduler Extension component.

1. Install and run the scheduler extension Docker container on the master node of the Kubernetes cluster.

	Run

        docker run -d --rm --name ritk8srdma-scheduler-entension --network host ritk8srdma/rit-k8s-rdma-scheduler-extender

	This will pull the Docker image for the scheduler extension from our Docker Hub repository and run it.

	If you would like to build the scheduler extension docker image yourself, the instructions are available within the [scheduler extension repository](https://github.com/rit-k8s-rdma/rit-k8s-rdma-scheduler-extender).

2. Modify the configuration of the core Kubernetes scheduler to register the scheduler extension.

	On the master node of the Kubernetes cluster, edit or add the file /etc/kubernetes/scheduler-policy-config.json to register the scheduler extension. The following entry should be added to the 'extenders' list within that file:

        {
            "urlPrefix": "http://127.0.0.1:8888/scheduler",
            "filterVerb": "rdma_scheduling",
            "bindVerb": "",
            "enableHttps": false,
            "nodeCacheCapable": false,
            "ignorable": false
        }

	Here, the IP address/port combination of 127.0.0.1 and 8888 is used because the scheduler extension is running on the same node as the core Kubernetes scheduler (the master node of the Kubernetes cluster), and listening on port 8888. If the extension is run elsewhere or listening on a different port, the 'urlPrefix' parameter should be editted accordingly.

	An example version of this file is available in the [scheduler extension repository](https://github.com/rit-k8s-rdma/rit-k8s-rdma-scheduler-extender/blob/master/kube_scheduler_config_files/scheduler-policy-config.json).


3. Ensure the core Kubernetes scheduler is using the configuration file where the scheduler extension is registered.

	Edit the file /etc/kubernetes/manifests/kube-scheduler.yaml on the master node of the Kubernetes cluster. Add the following volume to the pod definition if it does not exist (place the definition within the existing 'volumes' section if one exists):

        volumes:
        - hostPath:
            path: /etc/kubernetes/scheduler-policy-config.json
            type: FileOrCreate
          name: scheduler-policy-config

	Add the following directive to the command to be run in the kube-scheduler container:

        --policy-config-file=/etc/kubernetes/scheduler-policy-config.json

	Overall, the whole file should look like:

        apiVersion: v1
        kind: Pod
        metadata:
          annotations:
            scheduler.alpha.kubernetes.io/critical-pod: ""
          creationTimestamp: null
          labels:
            component: kube-scheduler
            tier: control-plane
          name: kube-scheduler
          namespace: kube-system
        spec:
          containers:
          - command:
            - kube-scheduler
            - --address=127.0.0.1
            - --kubeconfig=/etc/kubernetes/scheduler.conf
            - --policy-config-file=/etc/kubernetes/scheduler-policy-config.json
            - --leader-elect=true
            image: k8s.gcr.io/kube-scheduler:v1.13.5
            imagePullPolicy: IfNotPresent
            livenessProbe:
              failureThreshold: 8
              httpGet:
                host: 127.0.0.1
                path: /healthz
                port: 10251
                scheme: HTTP
              initialDelaySeconds: 15
              timeoutSeconds: 15
            name: kube-scheduler
            resources:
              requests:
                cpu: 100m
            volumeMounts:
            - mountPath: /etc/kubernetes/scheduler.conf
              name: kubeconfig
              readOnly: true
            - mountPath: /etc/kubernetes/scheduler-policy-config.json
              name: scheduler-policy-config
              readOnly: true
          hostNetwork: true
          priorityClassName: system-cluster-critical
          volumes:
          - hostPath:
              path: /etc/kubernetes/scheduler.conf
              type: FileOrCreate
            name: kubeconfig
          - hostPath:
              path: /etc/kubernetes/scheduler-policy-config.json
              type: FileOrCreate
            name: scheduler-policy-config
        status: {}

4. Ensure that the scheduler extension has started up correctly.

	Run `docker logs ritk8srdma-scheduler-entension` on the node where the scheduler extension is running. The output should include the following line at the top of its output:

        YYYY/MM/DD HH:MM:SS RDMA scheduler extender listening on port:  <port_number>

	This command can be run whenever necessary to view the logging output from the scheduler extension.


## CNI Plugin

This section covers the installation of the CNI plugin on each RDMA-enabled worker node in the Kubernetes cluster.

1. Install the CNI plugin executable.

	Copy the 'rit-k8s-rdma-cni-linux-amd64' executable from [the releases page of the repository](https://github.com/rit-k8s-rdma/rit-k8s-rdma-sriov-cni/releases). Place it in /opt/cni/bin/ on each RDMA-enabled worker node in the Kubernetes cluster.

2. Install the CNI plugin configuration file.

	Copy the file '10-ritk8srdma-cni.conf' from [the releases page of the repository](https://github.com/rit-k8s-rdma/rit-k8s-rdma-sriov-cni/releases). Place it in /etc/cni/net.d/ on each RDMA-enabled worker node in the Kubernetes cluster. This configuration file can be edited to fit the needs of your environment. Ensure that this file is the first one (lexicographically) within that directory. Kubernetes always uses the CNI configuration that comes first lexicographically within this directory.

	Within this file, the 'type' parameter specifies the name of the CNI executable that will be run when a pod is deployed. This name should match the name of the executable installed during step 1.

To compile the CNI plugin binary yourself, checkout the [CNI repository](https://github.com/rit-k8s-rdma/rit-k8s-rdma-sriov-cni). Enter the 'sriov' directory within the checked-out copy of the repository, then run `go install`. The binary should then be created in the appropriate Golang bin directory.

## Dummy Device Plugin

This section covers the installation of the Dummy Device Plugin onto all of the worker nodes in the Kubernetes cluster.

1. Use Kubernetes to deploy the dummy device plugin to all the nodes in the cluster.

	On the master node of the Kubernetes cluster, run:

        kubectl apply -f <dummy_plugin_yaml>

	Where <dummy_plugin_yaml\> is a YAML file that specifies the details of the Dummy Device Plugin. This file can be found at:

        https://github.com/rit-k8s-rdma/rit-k8s-rdma-dummy-device-plugin/blob/master/rdma-dummy-dp-ds.yaml

	Applying this configuration on the Kubernetes cluster will cause each worker node to pull container images built as part of this project from our [Docker Hub repository](https://hub.docker.com/u/ritk8srdma). These images contain the files necessary to run the Dummy Device Plugin.

	If you would like to build the Dummy Device Plugin Docker image yourself, the instructions are available within the [Dummy Device Plugin repository](https://github.com/rit-k8s-rdma/rit-k8s-rdma-dummy-device-plugin).

2. Verify that the Dummy Device Plugin is running on each worker node in the cluster.

	On the master node of the Kubernetes cluster, run:

        kubectl get pods -o wide --namespace kube-system

	Within the output for this command, you should see several lines with the name: `rdma-dummy-dp-ds-*` (one for each worker node in the cluster). The `status` column of each of these pods should show `Running`.

## Pod YAML Changes

	To take advantage of RDMA within a Kubernetes pod, that pod's definition (YAML file) will need to be updated to specify the RDMA interfaces that it requires. This involves the following steps:

1. Add the `rdma_interfaces_required` directive to the pod's metadata annotations:

        apiVersion: v1
        kind: Pod
        metadata:
          name: pod1
          annotations:
            rdma_interfaces_required: '[
                {"min_tx_rate": 15000, "max_tx_rate": 20000},
                {"min_tx_rate": 5000},
                {}
            ]'
        spec:
        ...

	The value of this annotation should be a JSON-formatted string that contains a list of RDMA interfaces needed by the pod, as well as the bandwidth limitations and reservations for each of those interfaces. In this case `min_tx_rate` specifies an amount of bandwidth that should be reserved for the pod to use exclusively through a specific RDMA interface, while `max_tx_rate` sets a cap on the amount of bandwidth that can used by a pod through an interface. Either or both of these properties can be omitted if you do not need a bandwidth cap/reservation. In the example above, three RDMA interfaces are requested by a pod: the first sets both properties, the second has only a bandwidth reservation, and the third has no limit nor any reserved bandwidth. The numbers used are in units of megabits of bandwidth per second (Mb/S).

2. Add a request for a `/dev/infiniband/` mount to each container that will need access to RDMA interfaces:

        ...
          containers:
          - image: mellanoxubuntudocker:latest
            name: mofed-test-ctr1
            resources:
              limits:
                rdma-sriov/dev-infiniband-mount: 1
        ...

	The line `rdma-sriov/dev-infiniband-mount: 1` indicates that the container requires privleged access to the `/dev/infiniband` directory. The quantity specified should be 1 (technically, the dummy device plugin is advertising an infinite amount of this resource type, but only one is needed to provide the mount point to the container).

3. Make sure the container images being deployed in the pod contain the necessary RDMA libraries. This can be done by utilizing Mellanox's 'mellanoxubuntudocker:latest' Docker container image (and/or using this as a base to build other containers).


An example of a complete pod configuration file is available from our [common repository](https://github.com/rit-k8s-rdma/rit-k8s-rdma-common/blob/master/example-pod-config.yml).


## Testing and Verification

### Deploy a Pod
	-deploy pod
	-kubectl get pods
	-if status != running:
		-kubectl describe pod
			-possible errors:
				-scheduler extension not running/found
					-see logs for kube-scheduler & scheduler extension itself
				-pod's rdma_interfaces_required JSON is malformatted
					-fix it
				-nodes in cluster did not respond to scheduler extension's requests
					-see scheduler extension's logs
					-make sure scheduler extension is contacting DaemonSet on same port daemonset is listening
					-use web browser/curl to make sure daemonset is working
				-nodes in cluster did not have enough free bandwidth
					-check response from daemonset using curl/browser
					-check how much bandwidth/how many interfaces the pod is requesting

### Test connectivity between two pods

	-have two pods w/ node selectors and run ib_send_bw between them

