# Proxmox VE Helpers

This repository is a set of scripts to better handle some of the Proxmox functions:

- automatically restart VMs on host suspend,
- allow to use CPU pinning,
- allow to set fifo scheduler
- allow to set affinity mask for vfio devices

Why to do CPU pinning?

- Usually, it is not needed as long as you don't use SMT
- If you use SMT, each vCPU is not equal, CPU pinning allows to ensure that VMs receive a real threads
- For having a good and predictable performance it is not needed to pin to exact cores, Linux can balance it very well
- In general the less we configure the better it works. These settings are hints to define affinity masks for resources.

## Installation

Clone and compile the repository:

```bash
# install dependencies
sudo apt-get install -f ruby ruby-dev rubygems build-essential
sudo gem install fpm
```

```bash
# compile pve-helpers
git clone https://github.com/aaraykay/pve-helpers
cd pve-helpers
sudo make install
```

## Usage

### 1. Enable snippet

You need to configure each machine to enable the hookscript.

The snippet by default is installed in `/var/lib/vz`
that for Proxmox is present as `local`.

```bash
qm set 204 --hookscript=local:snippets/exec-cmds
```

### 2. Configure VM

Edit VM description and add a new line if one or both these two commands.

### 2.1. `ph_pin_vcpus`

For the best performance you want to assign VM to physical cores,
not a mix of physical and virtual cores.

For example for `i7-8700` each core has two threads: 0+6, 1+7, 2+8.
You can easily check that with `lscpu -e`, checking which cores are
assigned twice.

```bash
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE MAXMHZ    MINMHZ
0   0    0      0    0:0:0:0       yes    4600.0000 800.0000
1   0    0      1    1:1:1:0       yes    4600.0000 800.0000
2   0    0      2    2:2:2:0       yes    4600.0000 800.0000
3   0    0      3    3:3:3:0       yes    4600.0000 800.0000
4   0    0      4    4:4:4:0       yes    4600.0000 800.0000
5   0    0      5    5:5:5:0       yes    4600.0000 800.0000
6   0    0      0    0:0:0:0       yes    4600.0000 800.0000
7   0    0      1    1:1:1:0       yes    4600.0000 800.0000
8   0    0      2    2:2:2:0       yes    4600.0000 800.0000
9   0    0      3    3:3:3:0       yes    4600.0000 800.0000
10  0    0      4    4:4:4:0       yes    4600.0000 800.0000
11  0    0      5    5:5:5:0       yes    4600.0000 800.0000
```

For example it is advised to assign one CPU less than the number of
physical cores. For the `i7-8700` it will be 5 cores.

For example we can pin each vCPU task for our first VM to its own non-physical thread:

```text
ph_pin_vcpus 7,8,9,10,11
```

Now for our second VM, we can pin each vCPU task to its own physical core. We deliberatly
choose to not assign `CORE 0`.

If you have two VMs concurrently running, you can assign it on one thread,
second on another thread, like this:

```text
VM 1:
ph_pin_vcpus 1,2,3,4,5

VM 2:
ph_pin_vcpus 7,8,9,10,11
```

### 2.2. use `vendor-reset` for fixing AMD Radeon reset bug

Instead of `ph_pci_unbind` and `ph_pci_rescan` install DKMS module from https://github.com/gnif/vendor-reset:

```bash
apt install dkms
git clone https://github.com/gnif/vendor-reset.git /usr/src/vendor-reset-0.1.1
dkms build vendor-reset/0.1.1
dkms install vendor-reset/0.1.1
echo vendor-reset >> /etc/modules
modprobe vendor-reset
```

### 2.3. `ph_set_haltpoll`

This setting changes the value of the kvm parameter `halt_poll_ns` in `/sys/module/kvm/parameters/halt_poll_ns`
Different configurations benefit from different settings. Default value is `20000`. In theory, a larger value would be beneficial for the performance/latency of a VM. 
In practice, most Ryzen systems work best with `halt_poll_ns` set to `0`.

Usage example:
```yaml
cat /etc/pve/qemu-server/110.conf

##Set halt_poll_ns
#ph_set_haltpoll 0
...
```

### 2.4. `ph_pin_irqs`

`ph_pin_irqs [--sleep=10s] [cpu cores] [--all] [interrupt name] [interrupt name...]`

This setting aims to simplify the process of assigning interrupts to the correct cpu cores in order to get the best performance
while doing a gpu/usb controller/audio controller passthrough. The goal is to have the same cores assigned to the VM using `ph_pin_vcpus`, 
be responsible for the interrupts generated by the devices that are fully passed through to the VM. 
This is very important for achieving the lowest possible latency and eliminating random latency spikes inside the VM. 
Ideally, you would also use something like irqbalance to move all other interrupts away from the VM assigned CPU cores and onto your other hypervisor-reserved cores. Same CPU mask can be used with irqbalance to have the VM cpu cores banned from getting any other interrupts.

Note: Isolating cpu cores with `isolcpus` while having its own small benefits, is not required to get these latency improvements.

An optional `--sleep=10s` can be assigned to modify
default `30s` wait duration.

The `--all` can be used to automatically assign interrupts of all configured `hostpci` devices.

Usage example:
```yaml
cat /etc/pve/qemu-server/110.conf
##CPU pinning
#ph_pin_vcpus 1-5
#ph_pin_irqs --sleep=10s 1-5 --all
...
```

In this particular use case, all interrupts with `vfio` in their name are assigned to cores `4,12,5,13,6,14,7,15,2,10,3,11`, which in term correspond to cores `2-7` and their SMT equivalents `10-15`.
In other words, cores `2,3,4,5,6,7` from an 8 core 3700x are assigned to the VM and to all of the interrupts from the GPU, the USB onboard controller, and the onboard audio controller.

### 2.5. `ph_qm_conflict` and `ph_qm_depends`

Sometimes some VMs are conflicting with each other due to dependency on the same resources,
like disks, or VGA.

There are helper commands to shutdown (the `ph_qm_conflict`) or start (the `ph_qm_depends`)
when main machine is being started.

```yaml
cat /etc/pve/qemu-server/204.conf

# ph_qm_conflict 204
# ph_qm_depends 207
...
```

This first `ph_qm_conflict` will shuttdown VM with VMID 204 before starting the current one,
and it will also start VMID 207, that might be a sibiling VM.

I use the `ph_qm_conflict` or `ph_qm_depends` to run Linux VM sometimes with VGA passthrough,
sometimes as a sibiling VM without graphics cards passed, but running in a console mode.

Be careful if you use `ph_pci_unbind` and `ph_pci_rebind`, they should be after the `ph_qm_*` commands.

### 2.6. `ph_pci_unbind` and `ph_pci_rebind`

It might be desirable to bind VGA to VM, but as soon as VM finishes
unbind that and allow to use on a host.

The `--all` can be used to unbind all devices.

The simplest is to ensure that VGA can render output on a host before
starting, then instruct Proxmox VE to unbind, and rebind devices:

```yaml
cat /etc/pve/qemu-server/204.conf

## Rebind VGA to host
#ph_pci_unbind 02 00 0
#ph_pci_unbind 02 00 1
#ph_pci_unbind --all
#ph_pci_rebind
```

### 3. Legacy features

These are features that are no really longer needed to achieve a good latency in a VM.

### 3.1. `ph_cpu_chrt` **no longer needed, outdated**

Running virtualized environment always results in quite random latency
due to amount of other work being done. This is also, because Linux
hypervisor does balance all threads that has bad effects on `DPC`
and `ISR` execution times. Latency in Windows VM can be measured with https://www.resplendence.com/latencymon. Ideally, we want to have the latency of `< 300us`.

To improve the latency you can switch to the usage of `FIFO` scheduler.
This has a catastrophic effects to everything else that is not your VM,
but this is likely acceptable for Gaming / daily use of passthrough VMs.

Configure VM description with:

```text
ph_cpu_chrt fifo 1
```

> Note:
> It seems that if Hyper-V entitlements (they are enabled for `ostype: win10`) are enabled this is no longer needed.
> I now have amazing performance without using `ph_cpu_chrt`.

### 3.2. `ph_pci_unbind` and `ph_pci_rescan` **no longer needed, outdated**

Just use `vendor-reset`.

There are multiple approaches to handle Radeon graphics cards. I did find that
to make it stable:

1. VGA bios needs to be exported, put in `/usr/share/kvm` and passed as `romfile` of `hostpci*`,
2. PCIE unbind/rescan needs to happen.

Exporting bios should happen ideally when running "natively", so with graphics card available,
ideally on Windows, with `GPU-Z`. Once bios is exported, you should ensure that it
contains UEFI section: https://pve.proxmox.com/wiki/Pci_passthrough#How_to_known_if_card_is_UEFI_.28ovmf.29_compatible.
Sometimes the bios can be found on https://www.techpowerup.com/vgabios/.
Ensure that you find the exact one for you `vid:pid` of your graphics card.

This is how my config looks like once a bios is put in a correct place:

```yaml
cat /etc/pve/qemu-server/204.conf

## Fix VGA
#ph_pci_rescan
#ph_pci_unbind 02 00 0
#ph_pci_unbind 02 00 1
...
hookscript: local:snippets/exec-cmds
...
hostpci0: 02:00,pcie=1,romfile=215895.rom,x-vga=1
...
machine: q35
...
```

The comment defines a commands to execute to unbind and rebind graphics card VM.

In cases where there are bugs in getting VM up, the `suspend/resume` cycle of Proxmox
helps: `systemctl suspend`.

### 4. Suspend/resume

There's a set of scripts that try to perform restart of machines
when Proxmox VE machine goes to sleep.

First, you might be interested in doing `suspend` on power button.
Edit the `/etc/systemd/logind.conf` to modify:

```text
HandlePowerKey=suspend
```

Then `systemctl restart systemd-logind.service` or reboot Proxmox VE.

After that every of your machines should restart alongside with Proxmox VE
suspend, thus be able to support restart on PCI passthrough devices,
like GPU.

**Ensure that each of your machines does support Qemu Guest Agent**.
This function will not work if you don't have Qemu Guest Agent installed
and running.

### 5. My setup

Here's a quick rundown of my environment that I currently use
with above quirks.

#### 5.1. Hardware

- i7-8700
- 48GB DDR4
- Intel iGPU used by Proxmox VE
- AMD RX560 2GB used by Linux VM
- GeForce RTX 2080 Super used by Windows VM
- Audio is being output by both VMs to the shared speakers that are connected to Motherboard audio card
- Each VM has it's own dedicated USB controller
- Each VM has a dedicated amount of memory using 1G hugepages
- Each VM does not use SMT, rather it is assigned to the thread 0 (Linux) or thread 1 (Windows) of each CPU, having only 5 vCPUs available to VM

#### 5.2. Kernel config

```text
GRUB_CMDLINE_LINUX=""
GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX pci_stub.ids=10de:1e81,10de:10f8,10de:1ad8,10de:1ad9,10de:13c2,10de:0fbb,1002:67ef,1002:aae0"
GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX intel_iommu=on kvm_intel.ept=Y kvm_intel.nested=Y i915.enable_hd_vgaarb=1 pcie_acs_override=downstream vfio-pci.disable_idle_d3=1"
GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX cgroup_enable=memory swapaccount=1"
GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX intel_pstate=disable"
GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX hugepagesz=1G hugepages=42"
```

#### 5.3. Linux VM

I use Linux for regular daily development work.

My Proxmox VE config looks like this:

```text
## CPU PIN
#ph_pin_vcpus 0-5
#ph_pin_irqs 0-5 --all
#
## Conflict (207 shares disks, 208 shares VGA)
#ph_qm_conflict 207
#ph_qm_conflict 208
agent: 1
args: -audiodev id=alsa,driver=alsa,out.period-length=100000,out.frequency=48000,out.channels=2,out.try-poll=off,out.dev=swapped -soundhw hda
balloon: 0
bios: ovmf
boot: dcn
bootdisk: scsi0
cores: 5
cpu: host
hookscript: local:snippets/exec-cmds
hostpci0: 02:00,romfile=215895.rom,x-vga=1
hostpci1: 04:00
hugepages: 1024
ide2: none,media=cdrom
memory: 32768
name: ubuntu19-vga
net0: virtio=32:13:40:C7:31:4C,bridge=vmbr0
numa: 1
onboot: 1
ostype: l26
scsi0: nvme-thin:vm-206-disk-1,discard=on,iothread=1,size=200G,ssd=1
scsi1: ssd:vm-206-disk-0,discard=on,iothread=1,size=100G,ssd=1
scsi10: ssd:vm-206-disk-1,iothread=1,replicate=0,size=32G,ssd=1
scsihw: virtio-scsi-pci
serial0: socket
sockets: 1
usb0: host=1050:0406
vga: none
```

#### 5.4. Windows VM

I use Windows for Gaming. It has dedicated RTX 2080 Super.

```text
## CPU PIN
#ph_pin_vcpus 6-11
#ph_pin_irqs 6-11 --all
agent: 1
args: -audiodev id=alsa,driver=alsa,out.period-length=100000,out.frequency=48000,out.channels=2,out.try-poll=off,out.dev=swapped -soundhw hda
balloon: 0
bios: ovmf
boot: dc
bootdisk: scsi0
cores: 5
cpu: host
cpuunits: 10000
efidisk0: nvme-thin:vm-204-disk-1,size=4M
hookscript: local:snippets/exec-cmds
hostpci0: 01:00,pcie=1,x-vga=1,romfile=Gigabyte.RTX2080Super.8192.190820.rom
hugepages: 1024
ide2: none,media=cdrom
machine: pc-q35-3.1
memory: 10240
name: win10-vga
net0: e1000=3E:41:0E:4D:3D:14,bridge=vmbr0
numa: 1
onboot: 1
ostype: win10
runningmachine: pc-q35-3.1
scsi0: ssd:vm-204-disk-2,discard=on,iothread=1,size=64G,ssd=1
scsi1: ssd:vm-204-disk-0,backup=0,discard=on,iothread=1,replicate=0,size=921604M
scsi3: nvme-thin:vm-204-disk-0,backup=0,discard=on,iothread=1,replicate=0,size=100G
scsihw: virtio-scsi-pci
sockets: 1
vga: none
```

#### 5.5. Switching between VMs

To switch between VMs:

1. Both VMs always run concurrently.
1. I do change the monitor input.
1. Audio is by default being output by both VMs, no need to switch it.
1. I use Barrier (previously Synergy) for most of time.
1. In other cases I have Logitech multi-device keyboard and mouse,
   so I switch it on keyboard.
1. I also have a physical switch that I use
   to change lighting and monitor inputs.
1. I have the monitor with PBP and PIP, so I can watch how Windows
   is updating while doing development work on Linux.

## Author, License

Kamil Trzciński, 2019-2021, MIT
