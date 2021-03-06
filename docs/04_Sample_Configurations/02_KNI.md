Intel® DPDK vSwitch supports the mapping of host-created Intel® DPDK objects into a Linux guest's network stack using Intel® DPDK Kernel Network Interface (KNI). This provides some of the performance you'd expect from the use of Intel® DPDK with the ability to use legacy socket-based applications.

This section contains instructions on how to compile and run a sample application that demonstrates performance of Intel® DPDK vSwitch with KNI integration. It also describes the additional configuration required for both host and client systems to use KNI.

______

## Overview

This guide covers a Phy->VM->Phy configuration using KNI vports:

```
                                                         __
    +--------------------------------------------------+   |
    | guest                                            |   |
    |                   IP forwarding                  |   |
    |              +-=------------------+              |   |
    |              |                    v              |   |
    |   +--------------+            +--------------+   |   |  guest
    |   |    vEth0     |            |    vEth1     |   |   |
    +---+--------------+------------+--------------+---+ __|
               ^                           :
               |                           |
               :                           v             __
    +---+--------------+------------+--------------+---+   |
    |   | ivshm vport1 |            | ivshm vport2 |   |   |
    |   +--------------+            +--------------+   |   |
    |          ^                           :           |   |
    |          |                           |           |   |  host
    |          :                           v           |   |
    |   +--------------+            +--------------+   |   |
    |   |   phy port   |  ovs_dpdk  |   phy port   |   |   |
    +---+--------------+------------+--------------+---+ __|
               ^                           :
               |                           |
               :                           v
    +--------------------------------------------------+
    |                                                  |
    |                traffic generator                 |
    |                                                  |
    +--------------------------------------------------+
```

______

## Initial Switch Setup

On the host, remove any configuration associated with a previous run of the switch:

```bash
pkill -9 ovs
rm -rf /usr/local/var/run/openvswitch/
rm -rf /usr/local/etc/openvswitch/
mkdir -p /usr/local/var/run/openvswitch/
mkdir -p /usr/local/etc/openvswitch/
rm -f /tmp/conf.db
```

Initialise the Open vSwitch database server:

```bash
./ovsdb-tool create /usr/local/etc/openvswitch/conf.db $OPENVSWITCH_DIR/vswitchd/vswitch.ovsschema
./ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock --remote=db:Open_vSwitch,manager_options &
```

Add a bridge to the switch:

```bash
./ovs-vsctl --no-wait add-br br0 -- set Bridge br0
  datapath_type=dpdk
```

Add ports to the bridge:

```bash
./ovs-vsctl --no-wait add-port br0 ovs_dpdk_1 -- set Interface ovs_dpdk_1
  type=dpdk ofport_request=1
./ovs-vsctl --no-wait add-port br0 ovs_dpdk_32 -- set Interface ovs_dpdk_32
  type=dpdk ofport_request=32
./ovs-vsctl --no-wait add-port br0 ovs_dpdk_33 -- set Interface ovs_dpdk_33
  type=dpdk ofport_request=33
```

Confirm the ports have been successfully added:

```bash
./ovs-vsctl show
```

Start `ovs_dpdk`:

```bash
./ovs_dpdk -c 0x0F -n 4 --proc-type primary --base-virtaddr=<virt_addr>
  -- -p 0x03 -n 2 -k 2 --stats=5 --vswitchd=0 --client-switching-core=1
  --config="(0,0,2),(1,0,3)
```

Start the Open vSwitch daemon:

```bash
./ovs-vswitchd -c 0x100 --proc-type=secondary -- --pidfile=/tmp/vswitchd.pid
```

______

## Flow Table Setup

Delete the default flow entries from the bridge:

```bash
./ovs-ofctl del-flows br0
```

Add flow entries to switch packets from `Port0` (16) to `KNI0` (32) on the ingress path, and from `KNI1` (33) to `Port1` (17) on the egress path:

```bash
./ovs-ofctl add-flow br0 in_port=16,dl_type=0x0800,nw_src=1.1.1.1,
  nw_dst=6.6.6.2,idle_timeout=0,action=output:32
./ovs-ofctl add-flow br0 in_port=33,dl_type=0x0800,nw_src=1.1.1.1,
  nw_dst=6.6.6.2,idle_timeout=0,action=output:17
```

______

## Guest Setup

### Copy Files

Copy files to the VM making sure the `-L` flag is used within `cp` to follow the symlinks.

```bash
rm -rf /tmp/qemu_share
mkdir -p /tmp/qemu_share
chmod 777 /tmp/qemu_share
mkdir -p /tmp/qemu_share/DPDK
mkdir -p /tmp/qemu_share/kni_client
cp -aL <DPDK_DIR>/* /tmp/qemu_share/DPDK
```

### Create IVSHMEM metadata

Run the IVSHMEM manager utility to create the metadata needed to be used with QEMU. In this example ports `KNI0` and `KNI1` are going to be shared over a metadata file named `vm_1`.

```bash
./ovs-ivshm-mngr/build/app/ovs-ivshm-mngr -c 0x1 --proc-type=secondary --
  vm_1:KNI0,KNI1
```

Among other information the utility will print out to `STDOUT` the exact IVSHMEM command line to be used when launching QEMU. Add this to the other QEMU arguments.

```bash
APP: QEMU cmdline for metadata 'vm_1': -device ivshmem,size=64M,shm=fd:/dev/hugepages/rtemap_0:0x0:0x200000:/dev/hugepages/rtemap_1:0xa00000:0x3200000:/dev/hugepages/rtemap_2:0x0:0x200000:/dev/zero:0x0:0x9fc000:/var/run/.dpdk_ivshmem_metadata_vm_1:0x0:0x4000
```

### Create VM

Start QEMU, for example:

```bash
./qemu/x86_64-softmmu/qemu-system-x86_64 -c 0x30 --proc-type secondary -n 4
  -- -cpu host -boot c -hda <qemu_imagename.qcow2> -snapshot -m 8192M -smp 2
  --enable-kvm -name 'client 1' -nographic -vnc :1 -pidfile /tmp/vm1.pid
  -drive file=fat:rw:/tmp/qemu_share,snapshot=off
  -monitor unix:/tmp/vm1monitor,server,nowait \
  -device ivshmem,size=64M,shm=fd:/dev/hugepages/rtemap_0:0x0:0x200000:
  /dev/hugepages/rtemap_1:0xa00000:0x3200000:
  /dev/hugepages/rtemap_2:0x0:0x200000:
  /dev/zero:0x0:0x9fc000:/var/run/.dpdk_ivshmem_metadata_vm_1:0x0:0x4000
```

______

## Guest Configuration

### Copy Files

Login to the VM and copy across the files

```bash
mkdir -p /mnt/kni
mount -o iocharset=utf8 /dev/sdb1 /mnt/kni
mkdir -p /root/kni
cp -a /mnt/kni/* /root/kni
```

### Build DPDK on the Guest

A small number of modifications to the standard Intel® DPDK driver are required to support KNI devices in the guest. These changes have been included as a patch. Apply the `rte_kni_module_1_6` patch before compiling Intel® DPDK:

```bash
export RTE_SDK=/root/kni/DPDK
export RTE_TARGET=x86_64-ivshmem-linuxapp-gcc
cd /root/kni/DPDK
patch -N -p1 < rte_kni_module_1_6.patch
make uninstall
make install T=x86_64-ivshmem-linuxapp-gcc
```

### Build `kni_client` App

``` bash
export RTE_SDK=/root/kni/DPDK
export RTE_TARGET=x86_64-ivshmem-linuxapp-gcc
cd /root/kni
cd kni_client
make clean
make
```

### Insert `rte_kni` Module

Once everything has been built, insert the (modified) `rte_kni` driver:

```bash
insmod /root/kni/DPDK/x86_64-ivshmem-linuxapp-gcc/kmod/rte_kni.ko
```

You can ensure that this has completed sucessfully by checking the output of `dmesg`, you should see something like the following:

```bash
dmesg
```

```bash
[ 2734.408833] KNI: ######## DPDK kni module loading ########
[ 2734.409226] KNI: loopback disabled
[ 2734.409239] KNI: IVSHMEM metadata found
[ 2734.409240] KNI: ######## DPDK kni module loaded ########
```

It is important to see the "IVSHMEM metadata found" message in the kernel log. If "IVSHMEM metadata NOT found" message is seen instead or if no IVSHMEM message is printed, uninstall all targets, re-apply the patch, and re-install again. An attempt to bring up a virtual interface without the IVSHMEM metadata being found will lead to a kernel panic on the guest.

### Execute `kni_client` App

The application accepts a port name parameter, which it uses to lookup the rings for the corresponding client. The port name has to match exactly with the one shared through the IVSHMEM manager utility. Having copied the application to the guest, and carrying out the setup steps described here, compile and run the `kni_client` application.

```bash
./build/kni_client -c 0x1 -n 4 -- -p KNI0 -p KNI1 &
```

### Configure Network Interfaces

Bring up virtual interfaces:

```bash
ifconfig vEth_KNI0 hw ether 00:4B:4E:49:30:00
ifconfig vEth_KNI1 hw ether 00:4B:4E:49:30:01
ifconfig vEth_KNI0 2.2.2.1/24 up
ifconfig vEth_KNI1 3.3.3.1/24 up
```

Turn on IP forwarding:

```bash
sysctl -w net.ipv4.ip_forward=1
```

Turn off reverse path filtering:

```bash
sysctl -w net.ipv4.conf.all.rp_filter=0
sysctl -w net.ipv4.conf.vEth_KNI0.rp_filter=0
```

Add a default gateway:

```bash
route add default gw 3.3.3.2 vEth_KNI1
```

Add an ARP entry to the default gateway:

```bash
arp -s 3.3.3.2 DE:AD:BE:EF:CA:FE
```

Check both virtual interfaces are up and running

```bash
ifconfig -a
```

______

## Run the Test

Start transmission of packets from the packet generator. If setup correctly, packets can be seen returning to the transmitting port from the DUT.

______
