To maximise throughput, assign individual cores to each of the various processes involved in the test setup. You can use either the `taskset` command or the core mask parameter passed to `ovs_client` and `ovs_dpdk` applications. Additionally, on the host, all available cores, with the exception of core 0, should be isolated from the kernel scheduler.

This section details a method to achieve maximum performance for an eight core system (sixteen logical cores if Intel® Hyper-Threading Technology enabled).

______

## Isolate Cores

In the host, edit `/boot/grub2/grub.cfg` (or `/etc/default/grub`, if applicable), specifically this line:

```
GRUBCMDLINELINUX="..."
```

Include the following:

```
isolcpus=1,2,...,n
```

**Note:** `n` should be the max number of physical cores available if Intel® Hyper-Threading Technology is enabled. We recommend that it is disabled. **Always leave core 0 for the operating system**.

Update the grub configuration file:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

You must then reboot your system for these changes to take effect.

______

## Command Line Affinitisation

For `ovs_dpdk`, substitute in this command:

```bash
./ovs_dpdk -c 0x0f -n 4 --proc-type=primary -- -n 3 -p 0x3 -k 2 --stats=1
  --vswitchd=0 -client_switching_core=1 --config="(0,0,2),(1,0,3)"
```

Then for `ovs_client`, substitute in this command:

```bash
./ovs_client -c 0x1 -n 4 -- -n 1
```

**Note:** For all Intel® DPDK-enabled applications, the core mask option (`-c`) must be set so that no two processes have overlapping core masks.

______

## Affinitising the Host Cores

| Process | Core | Core Mask | Comments |
|:-------:|:----:|:---------:|:--------:|
|Kernel           | 0 | 0x1 |All other CPUs isolated (`isolcpus` boot parameter)|
|client_switching_core  | 1 | 0x2 |Affinity set in `ovs_dpdk` command line |
|RX core          | 2 | 0x4 |Affinity set in `ovs_dpdk` command line |
|RX core          | 3 | 0x8 |Affinity set in `ovs_dpdk` command line |
|QEMU process VM1 VMCPU0 | 4 | 0x10 |`taskset -a <pid_of_qemu_process>` |
|QEMU process VM1 VMCPU1 | 5 | 0x20 |`taskset -a <pid_of_qemu_process>` |
|QEMU process VM2 VMCPU0 | 6 | 0x40 |`taskset -a <pid_of_qemu_process>` |
|QEMU process VM2 VMCPU1 | 7 | 0x80 |`taskset -a <pid_of_qemu_process>` |
