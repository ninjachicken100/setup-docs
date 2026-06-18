# openshift-local-gpu-passthrough

**Overview**
1. Setup Openshift Local 
2. Prepare Host for GPU Passthrough
3. Passthrough GPU to Openshift Local CRC VM
4. Install and Run NVIDIA GPU Operator

## 1. Setup Openshift Local

Download the latest CRC version from: 

https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc 

From the directory of crc, move it to the following url:

```
sudo mv crc /usr/local/bin
```


Log in to https://console.redhat.com/openshift/create/local and download the **pull secret**. You will be prompted to enter it upon running `crc start`


Run the following commands *( ~10mins)* :

```
crc setup
crc start
```


> [!TIP]
> run `crc console` via CLI to open console

> [!TIP]
> To use CLI to interact with Openshift local through kubeapi, you can enable `oc` commands by running: `eval $(crc oc-env)`

### 1.1. Configuring Openshift Local

Ensure that the crc instance is not running:


```
crc stop
```

Then run the following commands to configure the cpus, memory, and storage respectively:


```
crc config set disk-size <number of GiB> 
```
> [!TIP]
> Run `df -h | grep /home` to check for available disk space.


```
crc config set cpus <number of vCPUs>
```
> [!TIP]
>  Run `lscpu | grep -E "Model name|CPU\(s\)|Thread|Core|Socket"` and look for **Core(s) per socket:** to check for available CPU cores.


```
crc config set memory <memory in MiB>
```
> [!TIP]
> Run   `free -h` to check available RAM for allocation


To check if configured correctly:
```
crc config view
```


> [!NOTE]
> Min number for memory is 9216.


> [!TIP]
> You may use the following config 
> ```
> crc config set disk-size 500
> crc config set memory 49152
> crc config set cpus 24
> ```


## 2. Passthrough GPU to Openshift Local CRC VM

We have to first enable IOMMU on host device. To check, we run the following command

```
sudo virt-host-validate
```

Expected Output:
```
...
QEMU: Checking if IOMMU is enabled by kernel                               : WARN (IOMMU appears to be disabled in kernel. Add intel_iommu=on to kernel cmdline arguments)
...
```


Before we make the changes, we can check the pre-existing kernel arguments by running:


```
sudo grubby --info=DEFAULT
```

Expected Output:

```
index=0
kernel="/boot/vmlinuz-6.8.6-200.fc39.x86_64"
args="ro rootflags=subvol=root rhgb quiet amd_iommu=on"
root="UUID=b50d0db6-f6f9-4a04-92ee-59e0b7a7727f"
initrd="/boot/initramfs-6.8.6-200.fc39.x86_64.img"
title="Fedora Linux (6.8.6-200.fc39.x86_64) 39 (Workstation Edition)"
id="1bf6083ed0104139a7cd628f5754dcb3-6.8.6-200.fc39.x86_64"
```



To enable IOMMU, we disable the Nouveau driver by adding the following arguments to the kernel.

For **Intel** systems:

```
sudo grubby --args="rd.driver.blacklist=nouveau intel_iommu=on iommu=pt" --update-kernel DEFAULT
```

For **AMD** systems:

```
sudo grubby --args="rd.driver.blacklist=nouveau amd_iommu=on iommu=pt" --update-kernel DEFAULT
```

Check the kernel arguments again to see if they were appended properly by re-running:

```
sudo grubby --info=DEFAULT
```

For nouveau to be blacklisted, we have to run the following commands.

```
sudo sh -c 'printf "blacklist nouveau\noptions nouveau modeset=0\n" > /usr/lib/modprobe.d/blacklist-nouveau.conf'
```


Other than blacklisting `nouveau`, `vfio-pci` has to be explicitly set as the kernel driver. First, we grab the IDs 

``` lspci -nn | grep -i nvidia ```

Expected Output:

```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD104GLM [RTX 3500 Ada Generation Laptop GPU] [10de:27bb] (rev a1)
	Subsystem: Hewlett-Packard Company Device [103c:8ca7]
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau
01:00.1 Audio device [0403]: NVIDIA Corporation AD104 High Definition Audio Controller [10de:22bc] (rev a1)
	Subsystem: Hewlett-Packard Company Device [103c:8ca4]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
```

Add to Grubby args

``` sudo grubby --update-kernel=DEFAULT --args="vfio-pci.ids=10de:27bb,10de:22bc" ```


Create and populate /etc/dracut.conf.d/vfio.conf with the following contents:

```
sudo sh -c 'printf "add_drivers+=\" vfio vfio_iommu_type1 vfio_pci \"\nforce_drivers+=\" vfio vfio_iommu_type1 vfio_pci \"\n" > /etc/dracut.conf.d/vfio.conf'
```

Rebuild initramfs and reboot the machine 

```
sudo dracut -f --kver $(uname -r)
sudo reboot
```


After rebooting, check kernel drivers are reflecting `vfio-pci`:

```
lspci -nnk | grep -A3 -i nvidia
```

Expected Output:

```
...
	Kernel driver in use: vfio-pci
...
```


## 3. Passthrough the GPU to Openshift Local

Detach the GPU from the host so that it can be attached to Openshift local vm.

```
sudo virsh nodedev-dettach pci_0000_01_00_0
```

> [!IMPORTANT]
> Ensure cluster is not running by running `crc stop`


After detaching, make sure Openshift Local is not active by running `crc stop`.

Assign the GPU to Openshift Local by running:
```
sudo virsh edit crc
```

Enter the address information under `<devices> ... </devices>` like so:

```
...
<devices>
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <source>
      <address domain='0x0000' bus='0x01' slot='0x00' function='0x00'/>
    </source>
  </hostdev>
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <source>
      <address domain='0x0000' bus='0x01' slot='0x00' function='0x01'/>
    </source>
  </hostdev>
...
</devices>
```


You should have a warning, type `i`:
```
sudo virsh edit crc
setlocale: No such file or directory
error: XML document failed to validate against schema: Unable to validate doc against /usr/share/libvirt/schemas/domain.rng
Extra element devices in interleave
Element domain failed to validate content

Failed. Try again? [y,n,i,f,?]: 
y - yes, start editor again
n - no, throw away my changes
i - turn off validation and try to redefine again
f - force, try to redefine again
? - print this help
Failed. Try again? [y,n,i,f,?]: i
Domain crc XML configuration edited.
```

In CLI, run `virt-manager` 

Double-click into CRC. 

Under 'View', select 'Details' -> select 'Add Hardware' at bottom left of window -> PCI Host Device -> Select and add both NVIDIA card and audio device

## 5. Setup NVIDIA GPU Operator and Node Feature Discovery Operator

### 5.1. NVIDIA GPU Operator


In Openshift Console under OperatorHub/Software Catalog, search for **nvidia GPU Operator** and install the default version.

In Installed Operators, click into NVIDIA GPU Operator, create a cluster policy with MIG and MIG manager disabled.

> [!TIP]
> Disable MIG via 'Form view'


### 5.2. Node Feature Discovery Operator

In Openshift Console under OperatorHub/Software Catalog, search for Node Feature Discovery  and install the default version.

In Installed Operators, click into Node Feature Discovery and create a 'Node Feature Discovery' instance with default settings.

> [!NOTE]
> Upon setting up both Operators, the nvidia-gpu-operator namespace should start to spawn lots of pods after openshift-nfd namespace pods have spun and and are running (1/1).
