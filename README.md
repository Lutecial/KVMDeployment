# KVM Documentation (Installation & Opreations)

## Installation & Setup

### **_Install KVM on Bare Metal_**

First, install KVM and assorted tools:

```
sudo apt update
sudo apt install qemu-system-x86 qemu-kvm qemu libvirt-bin virt-manager virtinst bridge-utils cpu-checker virt-viewer
```

Then validate that that KVM was installed and that the CPU has VT-x virtualization enabled with kvm-ok.

```
$ sudo kvm-ok

INFO: /dev/kvm exists
KVM acceleration can be used
```

If you instead get a message that looks like below, then go in at the BIOS level and enable VT-x.

```
INFO: /dev/kvm does not exist
HINT: sudo modprobe kvm_intel
INFO: Your CPU supports KVM extensions
INFO: KVM (vmx) is disabled by your BIOS
HINT: Enter your BIOS setup and enable Virtualization Technology (VT),
and then hard poweroff/poweron your system
KVM acceleration can NOT be used
```

Then run the virt-host-validate utility to run a whole set of checks against your virtualization ability and KVM readiness.

```
sudo virt-host-validate
```

### **_Add User to Libvirt Groups_**

So that we can manage the guest VM as a regular user, we can add ourselves to all of the libvirt groups (e.g. libvirt, libvirt-qemu) and the kvm group.

```
cat /etc/group | grep libvirt | awk -F':' {'print $1'} | xargs -n1 sudo adduser $USER

# add user to kvm group also
sudo adduser $USER kvm

# relogin, then show group membership
exec su -l $USER
id | grep libvirt
```

Group membership requires a user to log back in, so if the “id” command does not show your libvirt\* group membership, logout and log back in, or try “exec su -l \$USER“.

### **_Create a Network Bridge between Host and VMs_**

By creating a bridged network, you can have guest VMs share the network connection of the Host machine.

As an operator, this means you need to use the tools provided at the OS level to create a network bridge. On older versions of Ubuntu, you would use 'brctl', but on bionic you use Netplan.

Go into the “/etc/netplan” directory and you should see a file named “01-netcfg.yaml” or “50-cloud-init.yml” that looks something like:

```
network:
  version: 2
  renderer: networkd

  ethernets:
    enp1s0:
      dhcp4: true
      dhcp6: false
      addresses: [192.168.1.239/24]
      gateway4: 192.168.1.1
      mtu: 1500
      nameservers:
        addresses: [8.8.8.8]
```

This is for a host with a single physical NIC, static IP of 192.168.1.239 on the 192.168.1.1 network and using Google DNS servers at 8.8.8.8.

To create a bridged network, you need to disable the specific settings on the physical network, and instead apply them to the bridge. Make a backup of your old file before modifying.

```
cd /etc/netplan

# make backup
sudo cp 50-cloud-init.yaml 50-cloud-init.yaml.orig

# modify, add bridge
sudo vi 50-cloud-init.yaml
```

Below is an example showing how we take the physical network example above, and created a bridge named ‘br0’ that has the same properties.

```
network:
  version: 2
  renderer: networkd

  ethernets:
    enp1s0:
      dhcp4: false
      dhcp6: false
      #addresses: [192.168.1.239/24]
      #gateway4: 192.168.1.1
      #mtu: 1500
      #nameservers:
      #  addresses: [8.8.8.8]

  bridges:
    br0:
      interfaces: [enp1s0]
      addresses: [192.168.1.239/24]
      gateway4: 192.168.2.1
      mtu: 1500
      nameservers:
        addresses: [8.8.8.8]
      parameters:
        stp: true
        forward-delay: 4
      dhcp4: no
      dhcp6: no
```

Then apply the new netplan with bridged network with the commands below. But be sure you have physical access to the host in case network connectivity needs to be investigated.

```
sudo netplan generate
sudo netplan --debug apply
```

You may temporarily lost network connectivity if you are connected over ssh after running the apply command.

You can see the network entities at the OS level by using these commands:

```
# bridge control
brctl show

# network control
networkctl
networkctl status br0

# ip list
ip a | grep " br0:" -A 3
# show host routes
ip route

# show arp table (IP to MAC)
arp -n
```

Now with the network bridge created at the OS level, you can configure a libvirt network to use this bridge. Create a file named “host-bridge.xml” containing the following:

```
<network>
  <name>host-bridge</name>
  <forward mode="bridge"/>
  <bridge name="br0"/>
</network>
```

Then create the network using these commands:

```
# create libvirt network using existing host bridge
virsh net-define host-bridge.xml
virsh net-start host-bridge
virsh net-autostart host-bridge

# state should be active, autostart, and persistent
virsh net-list --all
```

VM assigned to this network will be treated just like any other peer host. For example, if your upstream gateway router is DHCP enabled, new VM will be assigned an IP from that pool if the guest OS is configured to use DHCP. Likewise, the guest OS can be configured to use a static IP from the same subnet as the host bridge.

> _Optional  
> Although we don’t directly require it for NetPlan, install the network bridge utilities package for debugging._

```
sudo apt-get install bridge-utils -y
```

### **_Create KVM Storage Pool_**

Using the command pool-define-as with virsh to define new storage pool, you need also to specify name, type and type's arguments.

In our case, name will be Spool1, type will be dir. By default you could provide five arguments for the type:

- source-host
- source-path
- source-dev
- source-name
- target

For (Dir) type, we need the last argumet "target" to specify the path of storage pool, for the other arguments we could use "–" to unspecific them.

```
virsh pool-define-as Spool1 dir - - - - "/mnt/personal-data/SPool1/"
```

To check the all storage pools you have in the environment, use the following command. (newly created pool should display as "inactive")

```
virsh pool-list --all
```

Now it’s time to build the storage pool, which we have defined above with the following command.

```
virsh pool-build Spool1
```

Using the virsh command pool-start to active/enable the storage pool we have just created/built above.

```
virsh pool-start Spool1
```

Check the status of environment storage pools using the following command. (Spool1 should activated now)

```
virsh pool-list --all
```

Configure Spool1 to start by libvirtd service every time automaticlly.

```
virsh pool-autostart Spool1
```

Finally lets display information about our new storage pool.

```
virsh pool-info Spool1
```

KVM should be running properly by now and awaiting vms creation.

## Operational Notes:

### **_Create new virtual disk inside a pool_**:

```
qemu-img create -f qcow2 /data/kvm/pool/system.img 40G
```

### **_Create new linux vm_**:

```
virt-install --virt-type=kvm --name=webserver --vcpu=1 --ram=1024 --virt-type=kvm --hvm --os-variant ubuntu18.04 --cdrom ~/mini.iso --network network=host-bridge --graphics vnc --disk path=/data/kvm/pool/webserver.img,bus=virtio --noautoconsole
```

### **_Create new windows vm_**:

```
virt-install --virt-type=kvm --name=win10 --ram 8192 --vcpus=2 --virt-type=kvm --hvm --os-variant win10 --cdrom ~/17763.107.101029-1455.rs5_release_svc_refresh_CLIENTENTERPRISEEVAL_OEMRET_x64FRE_en-us.iso --cdrom ~/virtio-win-0.1.141.iso --network network=host-bridge --graphics spice --disk path=/data/kvm/pool/system.img,bus=virtio --noautoconsole
```

### **_Create new vm without specify a VDI (pool creation required)_**

```
--disk pool=kvmpool,size=25,bus=virtio,format=qcow2
```

## Troubleshoot

### **_Enable IOMMU_**

The solution to this issue is to enable IOMMU in your GRUB boot parameters.

You can do this by setting the following in /etc/default/grub

```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on"
```

If you're using an AMD processor, you should append amd_iommu=on to the boot parameters instead

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on"
```

Then run `update-grub` and reboot

### **_Enable & setup multi-monitor_**

type command: `virsh edit <my-guest-name>` and search for this section :

```
<video>
      <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'>
        <acceleration accel3d='no'/>
      </model>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0a' function='0x0'/>
    </video>
```

Replace heads='1' by heads='2' to enable dual screen.

### **_Method to change VM boot order and add new device_**:

The most common option is the BIOS bootloader where it allows you specifying boot order via changing domain XML. try command: `virsh edit <my-guest-name>` then adjust the boot order with this example (copied from libvirt.org):

```
<os>
  <type>hvm</type>
  <loader>/usr/lib/xen/boot/hvmloader</loader>
  <boot dev='network'/>
  <boot dev='cdrom'/>
  <boot dev='hd'/>
  <bootmenu enable='yes'/>
</os>
```

Which means the boot order is:

1. boot from network like PXE boot
2. boot from CD-ROM if no PXE boot
3. boot from local hard disk if no PXE boot nor bootable CD

## Reference

> [KVM installation on bare metal headless server (Pool creation incomplete)](https://fabianlee.org/2018/08/27/kvm-bare-metal-virtualization-on-ubuntu-with-kvm/)  
> [KVM Bridge network setup (a bit about VM creation)](https://fabianlee.org/2019/04/01/kvm-creating-a-bridged-network-with-netplan-on-ubuntu-bionic/)  
> [KVM instllation in brief](https://www.linuxtechi.com/install-configure-kvm-ubuntu-18-04-server/)  
> [KVM Pool Creation](https://www.tecmint.com/kvm-management-tools-to-manage-virtual-machines/)  
> [Configuration for IOMMU checkup and enable](https://serverfault.com/questions/633183/how-do-i-enable-kvm-device-passthrough-in-linux)  
> [X Server for windows to run linux app on bash on ubuntu on windows (for remote access virt-manager from server host)](https://seanthegeek.net/234/graphical-linux-applications-bash-ubuntu-windows/)  
> [Virtual maching creating windows related](http://www.geekpills.com/operating-system/linux/nstalling-windows-xp-kvm-platform-ubuntu-16-04-lts)  
> [Delete a storage pool](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/virtualization_administration_guide/delete-lvm-storage-pool-virsh)  
> [Add a new disk image to existing VMs](https://www.cyberciti.biz/faq/how-to-add-disk-image-to-kvm-virtual-machine-with-virsh-command/)  
> [Create, attach, detach disk to vm in kvm on command line](https://bgstack15.wordpress.com/2017/09/22/create-attach-detach-disk-to-vm-in-kvm-on-command-line/)  
> [List of os variants](http://thomasmullaly.com/2014/11/16/the-list-of-os-variants-in-kvm/)  
> [KVM cheatsheet](https://blog.programster.org/kvm-cheatsheet)  
> [Virtio driver for windows installtion](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/index.html)
