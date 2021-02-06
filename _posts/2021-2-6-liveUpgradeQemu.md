---
layout: post
title: Live Upgrade QEMU via Local Migration
---

QEMU, along with KVM (in most cases in industry), is being used as a thorough solution for virtualization by a lot of cloud providers. In such a combination, QEMU provides the virtualization of memory, block devices and so on. Once a QEMU virtual machine is started, it stands as a process, with its own address space, and this process is usually perceived as a PC/server to the people who are acessing it, there could be some important services, for example let's say a database server, running on such a virtual machine. 

Since there is a requirement for the service’s continuation sometimes, it is best to have the QEMU process keep running 24/7, but with the problem that all other softwares are facing as well, that is, as long as this software still has public attention, it always needs to be updated, new features would have to be added, bugs would have to be fixed. As long as the binary file should be updated, there must be a restart of the QEMU process, in other word, the restart of the virtual machine. Restarting a QEMU process could be quick, but rebooting the operating system inside it could possibly take a very long time. In this case, the continuation of the service cannot be guaranteed anymore.

So, having the virtual machine live upgrade its hypervisor (in our case, it is QEMU) is the key point to achieve that the VM’s runtime hypervisor is upgraded without the VM having to be restarted. 

Although, there is already a proposed solution for live updating QEMU in the upstream, as Oracle has proposed their method to [live update QEMU in KVM Summit 2020](https://blogs.oracle.com/linux/qemu-live-update). The usability of this function is that you should have the QEMU with their patches at first, however, many of the Virtual Machines running either in production are in older QEMU versions and some still use the Redhat’s version of qemu-kvm which may have not picked up those patches for live-upgrade. 

Therefore, another way to achieve the live upgrade with existing QEMU features is the exploitation of QEMU live migration. We all know that live migration has been a key feature of QEMU since very early version, it is a function that enables QEMU instance to be migrated from a host to another host without restarting the QEMU process, and it only takes very short downtime (usually 300ms by default, also configurable) for the vm. Below there is a very common live migration architecture between two hosts managed by libvirt. 

For the rest of the blog, I will use libvirt as the VM management tool to demonstrate how this would be implemented, but keep in mind that, libvirt is not indispensable for controlling QEMU instances.

<p align="center">
  <img width="600" src="/images/live_update_qemu/old_live_migration.png">
</p>

When the live migration is initiated, the destination libvirtd(libvirt daemon) starts the destination QEMU process using the QEMU binary in the destination host, and then the QEMU starts to transfer the memory of the source vm to the destination vm, as well as CPU state at the last point. It is easy to find out that if the newer version of QEMU is installed in the destination host, the QEMU instance after the migration should be with the newer QEMU version, thus, achieving the transition to the updated version without restarting the virtual machine. 

Apparently from the demonstration above, it can be easily inferred that: if the migration happens locally, a “live updating QEMU” then is achieved. That is true, although currently, libvirt does not support the local live migration of QEMU because of its own mechanisms, it is still not complicated to modify its code to support such operation, here is a proposed flow of operations:

<p align="center">
  <img width="300" src="/images/live_update_qemu/control_flow.png">
</p>


Here is the architecture of how each component interacts with each other.

<p align="center">
  <img width="600" src="/images/live_update_qemu/live_upgrade_qemu.png">
</p>

Libvirt, per se, does not support local migration of domains (in libvirt, a vm is called domain) because of its current  mechanism. But it is not hard to modify its code and adapt this feature into its mechanism. The primary changes that need to be made, besides adding a new open api, are temporarily modification the name of the destination domain during the local migration and permanently modification the domain’s UUID. 

[Here](https://gitlab.com/tianrenz2/libvirt/-/tree/live-upgrade) is the branch of libvirt that I have been working on to implement the local migration feature while the code is still being elaborated.

With no doubt, this method has some drawbacks:
  1. It requires to use another chunk of memory to hold the destination QEMU instance for the migration;
  2. It is possibly very slow when memory is suffering intensive writes because it keeps generating dirty pages (it actually depends on what strategy you are using for live migration, for example, enabling post-copy can avoid this).
  3. If using the libvirt, due to the limitation of libvirt’s mechanism, the domain’s UUID would have to be changed.
