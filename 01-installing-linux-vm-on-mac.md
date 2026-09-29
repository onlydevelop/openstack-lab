# Installing Linux VM on a Mac

## Purpose

This document records the setup used to create an Ubuntu Server ARM64
virtual machine on an Apple Silicon Mac for an OpenStack learning lab.

## Lab architecture

``` text
MacBook Air M4 / 24 GB RAM
        |
        v
      UTM
        |
        v
Ubuntu Server 24.04.5 LTS (ARM64)
        |
        v
   Kolla-Ansible
        |
        v
OpenStack services
```

## Why UTM

UTM was selected for a full Linux VM rather than a container-oriented
environment. The lab needs a normal Linux system on which Kolla-Ansible
and its Docker-based OpenStack services can run.

## VM configuration used

The VM was configured approximately as follows:

  Resource             Configuration
  -------------------- -------------------------------------
  Host                 MacBook Air M4
  Host RAM             24 GB
  Guest OS             Ubuntu Server 24.04.5 LTS
  Guest architecture   ARM64
  Virtualization       Native Apple Silicon virtualization
  vCPU                 8 cores
  RAM                  12--14 GB
  Main disk            100 GB
  Additional disk      40 GB
  Network adapters     2
  Network mode         Shared Network

The second 40 GB disk was later dedicated to the Cinder LVM backend.

## Install Ubuntu

1.  Download an Ubuntu Server ARM64 ISO.
2.  Create a new VM in UTM.
3.  Select Apple Silicon/ARM64 virtualization.
4.  Allocate the CPU, RAM and disk resources above.
5.  Attach the Ubuntu Server ARM64 ISO.
6.  Install Ubuntu Server.
7.  Install the OpenSSH server during the Ubuntu installation.
8.  Log in to the VM.

The VM used the hostname:

``` text
openstack
```

and the Linux user:

``` text
db
```

## Network layout

Two virtual NICs were used:

``` text
                 UTM Shared Network
                       |
          +------------+------------+
          |                         |
     enp0s1                       enp0s2
 Management                    External Neutron
 192.168.65.2                  interface
```

The management interface received:

``` text
192.168.65.2/24
```

The external interface was intentionally left without an IPv4 address.
Neutron uses it as the external network interface.

The management default route was:

``` text
default via 192.168.65.1 dev enp0s1
```

## OpenStack VIP

The Kolla internal VIP was configured as:

``` text
192.168.65.250
```

The VIP is assigned as a `/32` address on the management interface after
deployment.

## Host-to-VM access

UTM's Shared Network configuration used in this lab did not expose a GUI
port-forwarding option.

SSH tunneling was therefore used to access Horizon from macOS:

``` bash
ssh -L 8080:192.168.65.250:80 db@192.168.65.2
```

Then, from the Mac:

``` text
http://127.0.0.1:8080
```

The SSH session must remain open while using the tunnel.

## Useful verification commands

Check the management interface:

``` bash
ip addr show enp0s1
```

Check routes:

``` bash
ip route
```

Check connectivity:

``` bash
ping -c 3 192.168.65.1
```

Check the OpenStack VIP:

``` bash
curl -I http://192.168.65.250
```

A working Horizon endpoint returned:

``` text
HTTP/1.1 302 Found
location: http://192.168.65.250/auth/login/?next=/
```

## Important lab design decision

Bare Metal/Ironic was intentionally excluded from the initial lab. The
first stage focuses on core OpenStack services. Ironic can be added
later when a physical server with appropriate BMC/IPMI/Redfish
capabilities is available.
