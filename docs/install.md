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

3. Verify that the RDMA Hardware Daemon Set's REST endpoint is running.

	Once the status of each hardware daemon set pod, as inspected in the previous step, is `Running`, the REST endpoint should be available to query. During the normal operation of the system, this would be done by the scheduler extension during the scheduling of a new pod. However, we can perform this process manually using 'curl' or a web browser.

	First, determine the IP address or hostname of the worker node whose RDMA Hardware Daemon Set you would like to test. Then, in a web browser, navigate to:

        http://<IP_OR_HOSTNAME>:54005/getpfs

	Where <IP_OR_HOSTNAME\> is the IP or hostname of the worker node whose daemon set you would like to test, and 54005 is the port that daemon set is listening on (54005 is the default value at the time of writing).

	You should see output that resembles the following:

        [
            {
                "name":"enp4s0f0",
                "used_tx_rate":0,
                "capacity_tx_rate":100000,
                "used_vfs":0,
                "capacity_vfs":120,
                "vfs":[
                    {"vf":0, "mac":"7a:90:db:7b:30:ac", "vlan":0, "qos":0, "vlan_proto": "N/A", "spoof_check":"OFF", "trust":"ON", "link_state":"Follow", "min_tx_rate":0, "max_tx_rate":0, "vgt_plus":"OFF", "rate_group":0, "allocated":false},
                    {"vf":1, "mac":"c6:26:fe:e2:4e:95", "vlan":0, "qos":0, "vlan_proto":"N/A", "spoof_check":"OFF", "trust":"ON", "link_state":"Follow", "min_tx_rate":0, "max_tx_rate":0, "vgt_plus":"OFF", "rate_group":0, "allocated":false},
                    ...
                ]
            },
            {
                "name":"enp4s0f1",
                "used_tx_rate":0,
                "capacity_tx_rate":100000,
                "used_vfs":0,
                "capacity_vfs":120,
                "vfs":[
                    {"vf":0, "mac":"02:b9:9a:99:9e:ac", "vlan":0, "qos":0, "vlan_proto":"N/A", "spoof_check":"OFF", "trust":"ON", "link_state":"Follow", "min_tx_rate":0, "max_tx_rate":0, "vgt_plus":"OFF", "rate_group":0, "allocated":false},
                    {"vf":1, "mac":"ea:99:de:00:e8:8b", "vlan":0,"qos":0, "vlan_proto":"N/A", "spoof_check":"OFF", "trust":"ON", "link_state":"Follow", "min_tx_rate":0, "max_tx_rate":0, "vgt_plus":"OFF", "rate_group":0, "allocated":false},
                    ...
                ]
            },
            ...
        ]

	If the RDMA Hardware Daemon Set endpoint responds, and the hardware information presented in the list it returns accurately reflects the state and details of the RDMA hardware on its node, then it is working correctly.

	Respeat this process for every node in the cluster to verify that each instance of the daemon set is working correctly.


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

In order to verify that the system is working correctly, we can perform several tests. These tests will involve performing actions that exercise some components of the system, then performing checks to ensure those components functioned correctly.

### Deploy a Pod

In this test, we deploy a single pod within the cluster to ensure that basic connectivity exists between the scheduler extension, dummy device plugin, and CNI plugin.

1. Create a YAML file describing a pod to be deployed. Outfit this YAML file with the annotations mentioned in the 'Pod YAML Changes' section.

2. Request that Kubernetes deploy the pod onto the cluster. Running this command will indicate to Kubernetes that you would like to run the pod on one of the worker nodes in the cluster.

        kubectl create -f <pod_yaml_filename>

3. View the status of the pod. Running this command will inform you where in the deployment process the pod is currently.

        kubectl get pods -o wide

	Find the entry in the list that matches the name of the pod in your YAML file from step 1. You want the pod's status to be `Running`, though it may take several seconds to reach this state even when everything is working correctly, depending on the size of your cluster and the pod's resource requirements.

	If the pod does not reach the `Running` state after waiting for a while, you will need to perform some troubleshooting. First, observe more detailed information about the status of the pod by running:

        kubectl describe pod <POD_NAME>

	Where <POD_NAME\> is the name of the pod in your YAML file from step 1. Included in the information retrieved by this command is a the annotations used to request RDMA interface(s) for the pod, as well as the state of each container within the pod. When containers have a status other than `Running`, they will include a reason, such as `ErrImagePull`, which means the Docker image used to create the container couldn't be found on the node the pod was deployed on, or on Docker Hub. Also included in the output from this command is a list of 'Events' that have occured for the pod. If this events log contains a message like:

        Post http://127.0.0.1:8888/scheduler/rdma_scheduling: dial tcp 127.0.0.1:8888: connect: connection refused

	Then the scheduler extension could not be contacted by the core Kubernetes scheduler. Make sure the scheduler is running, and is configured to listen on the port the scheduler extension is trying to reach.

	Alternatively, if a message like the following appears in the event log:

        <N> RDMA Scheduler Extension: Unable to collect information on available RDMA resources for node.

	Then the scheduler extension experienced a timeout or a failed connection when attempting to contact the RDMA Hardware Daemon Set instanced on <N\> of the nodes in the cluster. If this occurs, repeat the verification steps from the RDMA Hardware Daemon Set installation section. If this works, then verify that the port and URL being requested from within the scheduler extension match those that the daemon set is listening on. It may be helpful to look at the scheduler extension logs in this case.

	Another possible message is the following:

        <N> RDMA Scheduler Extension: Node did not have enough free RDMA resources.

	Which means that the scheduler extension received information that indicated the available RDMA VFs/bandwidth on <N\> of the nodes in the cluster was not enough to satisfy the pod's requirements. If this seems incorrect, repeat step 3 of the RDMA Hardware Daemon Set install section, and inspect the amount of PFs, Bandwidth, and VFs available are correct for the node (and can satisfy what the pod is requesting).

	Yet another message that may show up in the events log for a pod is the following:

        <N> RDMA Scheduler Extension: 'rdma_interfaces_required' field in pod YAML file is malformatted.

	This simply means that the value in the `rdma_interfaces_required` annotation for the pod is not a valid JSON string. Simply delete the pending pod, fix the error, and re-deploy. It may be helpful to look at the scheduler extension logs in this case as well.

	An error like:

        <N> Insufficient rdma-sriov/dev-infiniband-mount.

	Comes from not having the dummy device plugin installed and configured correctly. View the status of its pods with `kubectl get pods -o wide --namespace kube-system`, and view the log files of each instance by executing `docker logs` on the right container on each worker node.

	Finally, a few other errors specific to our system may come from the CNI plugin. These will be marked as such, but are also unexpected if the CNI plugin is installed at all. In order to view the logs for a specific instance of our CNI plugin that appears to have an issue, search the relevant node's system log for messages that begin with `RIT-CNI`.

### Test connectivity between two pods

In order to ensure that the system has correctly provisioned pods with RDMA interfaces and set the correct bandwidth limits on those interfaces, we can run two pods on separate nodes in the Kubernetes cluster, then perform a bandwidth test between them.

1. Create two pod YAML files with the necessary information to request at least one RDMA interface. Add node selectors for this test to ensure that the two pods are deployed to two different nodes within the cluster (naturally, the nodes chosen should have enough RDMA resources available to satisfy the rquirements of the pods, the scheduler extension will prevent the pods from being deployed otherwise). Also, ensure the container images used for these pods contain the 'ib_send_bw' RDMA testing utility, as well as the necessary RDMA libraries.

2. Deploy the two pods onto their respective nodes in the cluster. Follow the instructions from the 'Deploy a Pod' section for troubleshooting.

3. On the node to which the first pod has been deployed, find the Docker container running within that pod using `docker ps`. Execute a shell within that container by running:

        docker exec -ti <CONTAINER_ID> /bin/bash

	Where <CONTAINER_ID\> is the ID of the container that you found by running `docker ps`.

4. While in a shell within that container, perform the following actions:

	Get the pods IP address (use the IP for the eth0 interface):

        ifconfig

	Get the name of the RDMA adapter for the VF that was allocated to the pod's eth0 interface:

        ibdev2netdev

	Run the receiving (server) side of the bandwidth testing application:

        ib_send_bw -d <RDMA_ADAPTER_NAME> -i 1 -F --report_gbits --run_infinitely

	Where <RDMA_ADAPTER_NAME\> is the name of the form 'mlx5_N' which was connected to the interface whose IP address you found when running `ifconfig`.

	This application is now waiting for an incoming connection. We will run the sending/client end of the test from a container within the other pod that we deployed onto another node in the cluster.

5. On the node to which the second pod has been deployed, find a Docker container running within that pod and execute a shell within it (follow the same process as you did for the first pod).

6. Within the shell running in the container from the second pod, perform the following actions:

	Get the name of the RDMA adapter for the VF that was allocated to the pod's eth0 interface:

        ibdev2netdev

	Run the sending (client) side of the bandwidth testing application:

        ib_send_bw -d <RDMA_ADAPTER_NAME> -i 1 -F --report_gbits <SERVER_IP> --run_infinitely

	Where <RDMA_ADAPTER_NAME\> is the name of the form 'mlx5_N' which was connected to the eth0 interface, and <SERVER_IP\> is the IP address you found for the interface within the other (first) pod in step 4.

	This command should display a table of bandwidth measurements that is updated periodically as it continues to run. The measurements displayed in the `BW average[Gb/sec]` column should all be at or below the bandwidth limit you set on the sending pod within its YAML file. If this is the case, then bandwidth limitation is working. A similar test can be run for bandwidth reservation, using additional pairs of pods that compete for bandwidth with the one that has the reservation.

