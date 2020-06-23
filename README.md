<h1 align="center">Welcome to KoolKubernetes SRIOV/DPDK Project 👋</h1>
<p>
  <a href="https://twitter.com/platform9sys" target="_blank">
    <img alt="Twitter: platform9sys" src="https://img.shields.io/twitter/follow/platform9sys.svg?style=social" />
  </a>
</p>

> This is a collection of Kubernetes SRIOV/DPDK Examples yaml files the are tested to work with Platform9 Free Teir Kubernetes.

## Author

👤 **Roopak Parikh**

* Twitter: [@platform9sys](https://twitter.com/platform9sys)
* Twitter: [@platform9sys](https://twitter.com/roopak_parikh)
* Github: [@pfray](https://github.com/roopakparikh)

## Show your support

Give a ⭐️ if this project helped you!

# Setup SRIOV DPDK
## Enable IOMMU
Enable IOMMU Support on all worker nodes
I/O Memory Management Unit (IOMMU) support is not enabled by default in the
CentOS* 7.0 distribution. IOMMU support is required for a VF to function properly when
assigned to a Virtual environment, such as a VM or a Container. The following kernel
boot parameter is required to enable IOMMU support for Linux kernels:

`intel_iommu=on iommu=pt`

Append to the GRUB_CMDLINE_LINUX entry in `/etc/default/grub` configuration
file.
```
$ cat /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DEFAULT=0
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto nomodeset rhgb quiet
intel_iommu=on iommu=pt hugepages=4096 pci=realloc pci=assignbusses"
GRUB_DISABLE_RECOVERY="true"
```

Update grub configuration using grub2-mkconfig
```
$ grub2-mkconfig -o /boot/grub2/grub.cfg
```
*Reboot* the machine for the IOMMU change to take effect.


## Configure Virtual Functions
Find the network interfaces
```
ip a


2: enp3s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether a0:36:9f:20:10:c8 brd ff:ff:ff:ff:ff:ff
    inet 10.128.237.203/26 brd 10.128.237.255 scope global dynamic enp3s0f0
       valid_lft 55643sec preferred_lft 55643sec
    inet6 fe80::a236:9fff:fe20:10c8/64 scope link
       valid_lft forever preferred_lft forever
```

And query the sriov functions provided by each interfaces

```
[centos@host-00 net]$ cat /sys/class/net/enp3s0f0/device/sriov_totalvfs
63
```

Now configure the number of virtual functions you want to test

```
echo 48 > /sys/class/net/enp3s0f0/device/sriov_numvfs
```

*Tip*: If you want to change the value always set the value to 0 first followed by the value you want to set it to

```
echo 0 > /sys/class/net/enp3s0f0/device/sriov_numvfs
echo 4 > /sys/class/net/enp3s0f0/device/sriov_numvfs

```

Now Virtual functions are up, by default this virtual functions are controlled by the kernel driver *ixgbevf* (check the actual word)

Check and load vfio_pci driver
```
# modprobe vfio_pci
# lsmod | grep vfio-pci
```

### Configure DPDK for the VFs

This section changes the VFs driver to be the user mode vfio driver instead of the default kernel mode driver.

Get dpdk-devbind.py from dpdk development kit

https://github.com/ceph/dpdk/blob/master/tools/dpdk-devbind.py


Find the PCI addresses for the various VFs. 

```
[root@host-02 MVNR_5GC_IMAGES]# lshw -c network -businfo | grep 'Virtual Function'
pci@0000:03:10.0                network        X550 Virtual Function
pci@0000:03:10.2                network        X550 Virtual Function
pci@0000:03:10.4                network        X550 Virtual Function
pci@0000:03:10.6                network        X550 Virtual Function
```

```
lshw -c network -businfo | grep 'Virtual Function' | awk '{print $1}' | grep -o '.......$'  > ./vf_pci.txt
cat ./vf_pci.txt | xargs ./dpdk-devbind.py -u 
cat ./vf_pci.txt | xargs ./dpdk-devbind.py  -b vfio-pci 
```

```
[root@host-02]# ./dpdk-devbind.py -s

Network devices using DPDK-compatible driver
============================================
0000:03:10.0 'X550 Virtual Function 1565' drv=vfio-pci unused=ixgbevf
0000:03:10.2 'X550 Virtual Function 1565' drv=vfio-pci unused=ixgbevf
0000:03:10.4 'X550 Virtual Function 1565' drv=vfio-pci unused=ixgbevf
0000:03:10.6 'X550 Virtual Function 1565' drv=vfio-pci unused=ixgbevf

Network devices using kernel driver
===================================
0000:03:00.0 'Ethernet Controller 10G X550T 1563' if=enp3s0f0 drv=ixgbe unused=vfio-pci *Active*
0000:03:00.1 'Ethernet Controller 10G X550T 1563' if=enp3s0f1 drv=ixgbe unused=vfio-pci
```

## Install Multus
Checkout multus

```
git clone https://github.com/intel/multus-cni.git
```

Create multus
```
kubectl apply -f ./multus-cni/images/multus-daemonset.yml
kubectl get pods -n kube-system
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-5dbf897646-wh2qj                 1/1     Running   1          47h
k8s-master-10.128.237.204                3/3     Running   3          47h
kube-multus-ds-amd64-cf9lr               1/1     Running   1          46h
kube-multus-ds-amd64-nh6h2               1/1     Running   0          46h
kube-multus-ds-amd64-sz68v               1/1     Running   0          22h
kube-multus-ds-amd64-xkhkd               1/1     Running   0          38h
```

The current __version__ is __3.4.1__


## Install SRIOV, SRIOV Device Plugin CNI

### Build (optional) sriov and sriovdp
Important, I ran into issues with golang = 1.6 ; had better luck with golang version 1.14. I also have prebuilt binaries checked in here (put a location)

Build Intel SR-IOV CNI
 ```
# git clone https://github.com/intel/sriov-cni.git
# cd sriov-cni
# make
# scp build/sriov root@<node-ip-addr>:/opt/cni/bin
```

Build SRIOV network device plugin
```
# git clone https://github.com/intel/sriov-network-device-plugin.git
# cd sriov-network-device-plugin
# make
# scp build/sriovdp root@<node-ip-addr>:/opt/cni/bin
```

### Copy sriov, sriovdp
Copy over these on ALL the worker nodes
```
scp sriov centos@10.128.240.223:/opt/cni/
scp sriovdp centos@10.128.240.223:/opt/cni/
```

## Configure SRIOV Device Plugin (SRIOV-DPDK)
This section is for SRIOV - DPDK specific

I used a specific release version but you can use any version of the sriov device plugin
```
git clone https://github.com/intel/sriov-network-device-plugin

```

### Find VF Info
Configure the sriovdp-config, this is the most *confusing* part of the whole excercise. 

First Obtain the list of the Virtual Functions
```
[root@host-02 MVNR_5GC_IMAGES]# lspci -nn | grep Virtual
00:11.0 PCI bridge [0604]: Intel Corporation C600/X79 series chipset PCI Express Virtual Root Port [8086:1d3e] (rev 06)
03:10.0 Ethernet controller [0200]: Intel Corporation X550 Virtual Function [8086:1565]
03:10.2 Ethernet controller [0200]: Intel Corporation X550 Virtual Function [8086:1565]
03:10.4 Ethernet controller [0200]: Intel Corporation X550 Virtual Function [8086:1565]
03:10.6 Ethernet controller [0200]: Intel Corporation X550 Virtual Function [8086:1565]
```

If you see the above the last portion of each Virtia; Function has the format `[<vendor-id>:<deviceid>]`

Also find the name of the physical NIC
```
[root@host-02 MVNR_5GC_IMAGES]# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp3s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether a0:36:9f:20:10:c8 brd ff:ff:ff:ff:ff:ff
    vf 0 MAC 00:00:00:00:00:00, spoof checking on, link-state auto, trust off, query_rss off
    vf 1 MAC 00:00:00:00:00:00, spoof checking on, link-state auto, trust off, query_rss off
    vf 2 MAC 00:00:00:00:00:00, spoof checking on, link-state auto, trust off, query_rss off
    vf 3 MAC 00:00:00:00:00:00, spoof checking on, link-state auto, trust off, query_rss off
```

### Configure SRIOV Device Plugin (k8s)
Here is the configured configMap.yaml that I used
```
$cat ./sriov-network-device-plugin/deployments/configMap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-system
data:
  config.json: |
    {
        "resourceList": [
            {
                "resourceName": "intel_sriov_dpdk",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["1565"],
                    "drivers": ["vfio-pci"],
                    "pfNames": ["enp3s0f0"]
                }
            }
        ]
    }

```

The github repo contains other examples in this example
1. pf9Names: Name of the Physical NIC
2. vendors: Name of the vendor (8086 is the VendorId, see last section)
3. drivers: Using vfio-pci: for vritual function 
4. devices: This is the device Id (see the last section value 1565 deviceId)

Apply Kubernetes Manifests

```
#kubectl apply -f ./sriov-network-device-plugin/deployments/configMap.yaml
#kubectl apply -f ./sriov-network-device-plugin/deployments/k8s-v1.16/sriovdp-daemonset.yaml
#kubectl --kubeconfig ../k get pods -n kube-system | grep sriov
kube-sriov-device-plugin-amd64-5lzxb     1/1     Running   0          24h
kube-sriov-device-plugin-amd64-86xd4     1/1     Running   0          45h
kube-sriov-device-plugin-amd64-ggphr     1/1     Running   0          39h
kube-sriov-device-plugin-amd64-jxrg7     1/1     Running   1          45h
```

Check Nodes
```
kubectl --kubeconfig ./k describe node 10.128.237.203
Name:               10.128.237.203
Roles:              worker
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    infra=true
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=10.128.237.203
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/worker=
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 17 Jun 2020 22:10:16 -0700
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  10.128.237.203
  AcquireTime:     <unset>
  RenewTime:       Fri, 19 Jun 2020 22:33:59 -0700
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Fri, 19 Jun 2020 22:33:11 -0700   Thu, 18 Jun 2020 11:54:58 -0700   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Fri, 19 Jun 2020 22:33:11 -0700   Thu, 18 Jun 2020 11:54:58 -0700   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Fri, 19 Jun 2020 22:33:11 -0700   Thu, 18 Jun 2020 11:54:58 -0700   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Fri, 19 Jun 2020 22:33:11 -0700   Thu, 18 Jun 2020 11:54:58 -0700   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.128.237.203
  Hostname:    10.128.237.203
Capacity:
  cpu:                         32
  ephemeral-storage:           109872976Ki
  hugepages-1Gi:               0
  hugepages-2Mi:               2Gi
  intel.com/intel_sriov_dpdk:  4
  memory:                      115500392Ki
```

The SRIOV device plugin is advertising the name intel_sriov_dpdk based on the configMap.yaml (remember to use this name going forward)

### Configure Network Definition
Now it is time to create NetworkAttachmentDefinition required by Multus

create a new NetworkAttachmentDefinition

```
#cat sriov-dpdk-net.yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-dpdk-net-1
  annotations:
    k8s.v1.cni.cncf.io/resourceName: intel.com/intel_sriov_dpdk
spec:
  config: '{
    "type": "sriov"
  }'
# kubectl apply -f ./sriov-dpdk-net.yaml
# kubectl get net-attach-def
NAME               AGE
sriov-dpdk-net-1   31h
```

## Run a sample application
Here is a sample application, the application is shamelessly copied from redhats app-netutil repository  https://github.com/openshift/app-netutil/

```
apiVersion: v1
kind: Pod
metadata:
  name: testpod3
  annotations:
    k8s.v1.cni.cncf.io/networks:  sriov-dpdk-net-1
spec:
  containers:
  - name: testpod3
    image: rparikh/dpdk-app-centos
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /etc/podnetinfo
      name: podnetinfo
      readOnly: false
    resources:
      requests:
        memory: 1Gi
        #cpu: "4"
        intel.com/intel_sriov_dpdk: 2
      limits:
        #cpu: "4"
        intel.com/intel_sriov_dpdk: 2
    # Uncomment to control which DPDK App is running in container.
    # If not provided, l3fwd is default.
    #   Options: l2fwd l3fwd testpmd
    #env:
    #- name: DPDK_SAMPLE_APP
    #  value: "l2fwd"
    #
    # Uncomment to debug DPDK App or to run manually to change
    # DPDK command line options.
    command: ["sleep", "infinity"]
  volumes:
  - name: podnetinfo
    downwardAPI:
      items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "annotations"
          fieldRef:
            fieldPath: metadata.annotations
  
```

Apply it

```
# kubectl apply -f ./testpods3.yaml
```

```
kubectl --kubeconfig ./k exec -it  testpod3 bash
[root@testpod3 /]# dpdk-app
ENTER dpdk-app:
 argc=1
 dpdk-app
  cpuRsp.CPUSet = 0-31
  Interface[0]:
    IfName="eth0"  Name="containernet"  Type=SR-IOV
    MAC="82:34:02:37:38:26"  IP="10.20.85.217"
    PCIAddress=0000:03:10.4
  Interface[1]:
    IfName="net1"  Name="sriov-network"  Type=SR-IOV
    MAC=""
    PCIAddress=0000:03:10.6
 myArgc=15
 dpdk-app -n 4 -l 1 --master-lcore 1 -w 0000:03:10.4 -w 0000:03:10.6 -- -p 0x3 -P --config="(0,0,1),(1,0,1)" --parse-ptype
EAL: Detected 32 lcore(s)
EAL: Detected 2 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'VA'
EAL: No available hugepages reported in hugepages-1048576kB
EAL: 1024 hugepages of size 2097152 reserved, but no mounted hugetlbfs found for that size
EAL: FATAL: Cannot get hugepage information.
EAL: Cannot get hugepage information.
EAL: Error - exiting with code: 1
  Cause: Invalid EAL parameters
```

Also note the envrionmental variables
```
[root@testpod3 /]# set | grep PCI
PCIDEVICE_INTEL_COM_INTEL_SRIOV_DPDK=0000:03:10.4,0000:03:10.6
```

# Reference

* https://builders.intel.com/docs/networkbuilders/enabling_new_features_in_kubernetes_for_NFV.pdf
* https://docs.google.com/document/d/1WRBo_Eajzh3zXB3doGMmOODZa4btBa3pvyJAESa6qJM/edit
* https://builders.intel.com/docs/networkbuilders/adv-network-features-in-kubernetes-app-note.pdf
* https://github.com/openshift/app-netutil/blob/master/samples/dpdk_app/sriov/README.md
* https://doc.dpdk.org/guides/tools/pmdinfo.html
