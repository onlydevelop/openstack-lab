# Setting Up OpenStack in the Linux VM

## Purpose

This document records the OpenStack installation performed inside the
Ubuntu Server ARM64 VM using Kolla-Ansible.

## Final architecture

``` text
MacBook Air M4 / 24 GB
        |
        v
       UTM
        |
        v
Ubuntu Server 24.04.5 ARM64
        |
        +-----------------------------+
        |                             |
        |       Kolla-Ansible         |
        |              |              |
        |              v              |
        |       Docker containers     |
        |                             |
        |  +-----------------------+  |
        |  | Keystone              |  |
        |  | Horizon               |  |
        |  | Glance                |  |
        |  | Nova                  |  |
        |  | Neutron               |  |
        |  | Cinder                |  |
        |  | Placement             |  |
        |  | Heat                  |  |
        |  | Swift                 |  |
        |  +-----------------------+  |
        |                             |
        +-----------------------------+
```

## 1. Create the Kolla-Ansible virtual environment

Inside Ubuntu:

``` bash
python3 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate
pip install -U pip
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2026.1
```

The installed Kolla-Ansible version during this setup was:

``` text
22.2.1.dev11
```

## 2. Prepare Kolla configuration

The Kolla configuration directory was:

``` text
/etc/kolla
```

It contained:

``` text
globals.yml
passwords.yml
```

The all-in-one inventory was:

``` text
~/openstack/all-in-one
```

Generate passwords:

``` bash
kolla-genpwd
```

## 3. Configure globals.yml

The important configuration used was:

``` yaml
kolla_base_distro: "ubuntu"
openstack_tag_suffix: "-aarch64"

kolla_internal_vip_address: "192.168.65.250"

network_interface: "enp0s1"
neutron_external_interface: "enp0s2"

enable_cinder: true
enable_cinder_backend_lvm: true
enable_swift: true
enable_horizon: "{{ enable_openstack_core | bool }}"
```

The ARM64 suffix is important because the VM runs on Apple Silicon:

``` yaml
openstack_tag_suffix: "-aarch64"
```

## 4. Fix hostname resolution

Kolla's RabbitMQ precheck required the hostname to resolve to the VM's
management IP rather than the loopback address.

The effective mapping was:

``` text
192.168.65.2 openstack
```

Verify with:

``` bash
getent hosts openstack
```

## 5. Install Docker

Ubuntu's Docker package was installed:

``` bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

The user was added to the Docker group:

``` bash
sudo usermod -aG docker db
```

After refreshing the login/session, verify:

``` bash
docker ps
```

Kolla-Ansible also required the Python Docker SDK in the Kolla virtual
environment:

``` bash
source ~/kolla-venv/bin/activate
pip install docker
```

## 6. Configure passwordless sudo for Ansible

A sudoers entry was configured for the `db` user:

``` text
db ALL=(ALL) NOPASSWD: ALL
```

Verify:

``` bash
ansible localhost -i ./all-in-one -m command -a "whoami" -b
```

Expected result:

``` text
root
```

## 7. Configure Cinder LVM storage

A second 40 GB virtual disk was added to the UTM VM.

It appeared as:

``` text
/dev/vdb
```

Create the physical volume:

``` bash
sudo pvcreate /dev/vdb
```

Create the volume group:

``` bash
sudo vgcreate cinder-volumes /dev/vdb
```

Verify:

``` bash
sudo pvs
sudo vgs
```

Do not create individual logical volumes manually. Cinder creates volume
LVs dynamically.

## 8. Run Kolla prechecks

The prechecks initially required the test-image option:

``` bash
kolla-ansible prechecks -i ./all-in-one --use-test-images
```

The prechecks eventually completed successfully:

``` text
localhost ok=116 changed=0 unreachable=0 failed=0 skipped=157
```

In this Kolla-Ansible version, `--use-test-images` is accepted by the
precheck path but not by `deploy`.

## 9. Bootstrap the server

Run:

``` bash
kolla-ansible bootstrap-servers -i ./all-in-one
```

The bootstrap completed successfully:

``` text
localhost ok=45 changed=7 unreachable=0 failed=0 skipped=30
```

## 10. Pull the container images

Before deployment, pull the Kolla images:

``` bash
kolla-ansible pull -i ./all-in-one
```

This succeeded.

The resulting images used the ARM64 tag, for example:

``` text
quay.io/openstack.kolla/kolla-toolbox:2026.1-ubuntu-noble-aarch64
quay.io/openstack.kolla/nova-compute:2026.1-ubuntu-noble-aarch64
quay.io/openstack.kolla/neutron-server:2026.1-ubuntu-noble-aarch64
quay.io/openstack.kolla/cinder-volume:2026.1-ubuntu-noble-aarch64
```

The toolbox image was particularly important:

``` text
quay.io/openstack.kolla/kolla-toolbox:2026.1-ubuntu-noble-aarch64
```

## 11. Deploy OpenStack

Run:

``` bash
kolla-ansible deploy -i ./all-in-one
```

The successful deployment ended with:

``` text
localhost ok=468 changed=203 unreachable=0 failed=0 skipped=298 rescued=0 ignored=1
```

## 12. Verify containers

Run:

``` bash
docker ps --format 'table {{.Names}}\t{{.Status}}' | sort
```

The successful deployment showed healthy services including:

``` text
cinder_api
cinder_backup
cinder_scheduler
cinder_volume

glance_api

keystone
keystone_fernet
keystone_ssh

nova_api
nova_compute
nova_conductor
nova_libvirt
nova_metadata
nova_novncproxy
nova_scheduler
nova_ssh

neutron_server
neutron_dhcp_agent
neutron_l3_agent
neutron_metadata_agent
neutron_openvswitch_agent

placement_api

horizon

mariadb
rabbitmq
memcached
haproxy
openvswitch_db
openvswitch_vswitchd

kolla_toolbox
```

Check the toolbox explicitly:

``` bash
docker ps --filter name=kolla_toolbox
```

## 13. Verify Horizon internally

From Ubuntu:

``` bash
curl -I http://192.168.65.250
```

The working result was:

``` text
HTTP/1.1 302 Found
location: http://192.168.65.250/auth/login/?next=/
```

This confirms that HAProxy/Horizon is responding through the Kolla VIP.

## 14. Obtain the admin password

``` bash
grep '^keystone_admin_password:' /etc/kolla/passwords.yml
```

Horizon credentials:

``` text
Username: admin
Password: value of keystone_admin_password
```

## 15. Access Horizon from the Mac

Because the UTM Shared Network did not provide a GUI port-forwarding
option, use an SSH tunnel:

``` bash
ssh -L 8080:192.168.65.250:80 db@192.168.65.2
```

Keep that SSH session open.

On macOS, browse to:

``` text
http://127.0.0.1:8080
```

The tunnel was verified with:

``` bash
curl -I http://127.0.0.1:8080
```

which returned:

``` text
HTTP/1.1 302 Found
location: http://127.0.0.1:8080/auth/login/?next=/
```

The OpenStack installation was therefore functioning correctly.

## Core services in this lab

``` text
Keystone    Identity
Horizon     Dashboard
Nova        Compute
Neutron     Networking
Glance      Images
Cinder      Block storage
Swift       Object storage
Placement   Resource placement
Heat        Orchestration
RabbitMQ    Messaging
MariaDB     Database
HAProxy     VIP/load balancing
OVS         Virtual switching
```
