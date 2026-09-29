# OpenStack Lab Troubleshooting

This document records the problems encountered while installing
OpenStack with Kolla-Ansible in an Ubuntu ARM64 VM on Apple Silicon,
together with the resolution used.

## 1. Kolla RabbitMQ hostname precheck failure

### Symptom

Kolla prechecks complained about hostname/RabbitMQ resolution.

### Cause

The hostname was resolving to the loopback address instead of the VM's
management address.

### Resolution

Make the hostname resolve to the management IP:

``` text
192.168.65.2 openstack
```

Verify:

``` bash
getent hosts openstack
```

The hostname should resolve to `192.168.65.2`.

------------------------------------------------------------------------

## 2. Cinder enabled without a backend

### Symptom

Kolla prechecks failed after Cinder was enabled.

### Cause

Cinder requires a configured backend.

### Resolution

Enable the LVM backend:

``` yaml
enable_cinder: true
enable_cinder_backend_lvm: true
```

Add a second VM disk and create the Cinder volume group:

``` bash
sudo pvcreate /dev/vdb
sudo vgcreate cinder-volumes /dev/vdb
```

Verify:

``` bash
sudo pvs
sudo vgs
```

Do not create Cinder volume LVs manually.

------------------------------------------------------------------------

## 3. Docker socket failure during bootstrap

### Symptom

Kolla bootstrap caused Docker to fail with messages similar to:

``` text
Socket unit configuration has changed while unit has been running
```

and:

``` text
failed to load listeners: no sockets found via socket activation
```

### Cause

Kolla changed Docker's socket configuration while `docker.socket` was
already active.

### Resolution

Stop and reset the socket/service state:

``` bash
sudo systemctl stop docker.service
sudo systemctl stop docker.socket
sudo systemctl reset-failed docker.socket docker.service
sudo systemctl start docker.socket
sudo systemctl start docker.service
```

Verify:

``` bash
sudo systemctl is-active docker.socket
sudo systemctl is-active docker.service
docker ps
```

Then rerun:

``` bash
kolla-ansible bootstrap-servers -i ./all-in-one
```

The bootstrap subsequently succeeded.

------------------------------------------------------------------------

## 4. `--use-test-images` rejected by deploy

### Symptom

This command failed:

``` bash
kolla-ansible deploy -i ./all-in-one --use-test-images
```

with:

``` text
kolla-ansible deploy: error: unrecognized arguments: --use-test-images
```

### Cause

In the installed Kolla-Ansible version, `--use-test-images` is available
for the relevant precheck path but is not a valid `deploy` argument.

### Resolution

Do not add the option to `deploy`.

Use:

``` bash
kolla-ansible deploy -i ./all-in-one
```

For prechecks, the option was accepted:

``` bash
kolla-ansible prechecks -i ./all-in-one --use-test-images
```

------------------------------------------------------------------------

## 5. `kolla_toolbox` missing during deployment

### Symptom

Deployment stopped at:

``` text
TASK [nova-cell : List all hypervisors]
```

with:

``` text
kolla_toolbox container is missing or not running!
```

Initial checks showed:

``` bash
docker ps -a --filter name=kolla_toolbox
```

returned no container.

Also:

``` bash
docker images
```

showed no local Kolla images.

### Cause

The required Kolla container images had not yet been pulled.

### Resolution

Run:

``` bash
kolla-ansible pull -i ./all-in-one
```

Verify:

``` bash
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
```

The important image appeared as:

``` text
quay.io/openstack.kolla/kolla-toolbox:2026.1-ubuntu-noble-aarch64
```

Then run:

``` bash
kolla-ansible deploy -i ./all-in-one
```

Deployment succeeded.

### Lesson

When Kolla reports that a service container is missing, first determine
whether the image exists locally before attempting to create or repair
the container manually.

------------------------------------------------------------------------

## 6. ARM64 image configuration

### Symptom

The VM is running on Apple Silicon, so standard x86_64 images are
inappropriate.

### Configuration

The Kolla configuration used:

``` yaml
openstack_tag_suffix: "-aarch64"
```

The resulting images were tagged:

``` text
2026.1-ubuntu-noble-aarch64
```

Examples:

``` text
nova-compute
neutron-server
cinder-volume
kolla-toolbox
```

### Verification

``` bash
docker images --format '{{.Repository}}:{{.Tag}}' | sort
```

------------------------------------------------------------------------

## 7. `kolla-ansible status` is not available

### Symptom

This command failed:

``` bash
kolla-ansible status -i ./all-in-one
```

with:

``` text
kolla-ansible: 'status -i ./all-in-one' is not a kolla-ansible command
```

### Resolution

Do not use `status` with this installed Kolla-Ansible version.

Use Docker directly:

``` bash
docker ps --format 'table {{.Names}}\t{{.Status}}' | sort
```

For example:

``` bash
docker ps --filter name=kolla_toolbox
```

------------------------------------------------------------------------

## 8. `genconfig` unexpectedly reached deployment tasks

### Symptom

Running:

``` bash
kolla-ansible genconfig -i ./all-in-one
```

in this environment still reached the site playbook and eventually
failed at:

``` text
TASK [nova-cell : List all hypervisors]
```

### Resolution

Do not rely on `genconfig` as a safe, isolated diagnostic step in this
particular installation.

Use the actual image pull operation:

``` bash
kolla-ansible pull -i ./all-in-one
```

and inspect the resulting Docker images.

------------------------------------------------------------------------

## 9. Horizon works inside the VM but not directly from macOS

### Symptom

Inside Ubuntu:

``` bash
curl -I http://192.168.65.250
```

returned:

``` text
HTTP/1.1 302 Found
```

but Firefox on the Mac could not connect directly to:

``` text
http://192.168.65.250
```

### Cause

UTM Shared Network provides guest networking through its virtual NAT
network. The Mac did not have direct routing to the guest's
`192.168.65.0/24` network.

The UTM Network settings used in this lab did not expose a GUI
port-forwarding configuration.

### Resolution

Use an SSH tunnel from the Mac:

``` bash
ssh -L 8080:192.168.65.250:80 db@192.168.65.2
```

Then access:

``` text
http://127.0.0.1:8080
```

### Verification

From a second Mac terminal:

``` bash
curl -I http://127.0.0.1:8080
```

The successful result was:

``` text
HTTP/1.1 302 Found
location: http://127.0.0.1:8080/auth/login/?next=/
```

The tunnel was therefore working.

------------------------------------------------------------------------

## 10. Browser showed `503 Service Unavailable` through the tunnel

### Symptom

Firefox initially showed:

``` text
503 Service Unavailable

No server is available to handle this request.
```

### Investigation

Test the tunnel directly:

``` bash
curl -I http://127.0.0.1:8080
```

Result:

``` text
HTTP/1.1 302 Found
location: http://127.0.0.1:8080/auth/login/?next=/
```

Also test with the OpenStack VIP as the HTTP Host header:

``` bash
curl -I -H 'Host: 192.168.65.250' http://127.0.0.1:8080
```

Result:

``` text
HTTP/1.1 302 Found
location: http://192.168.65.250/auth/login/?next=/
```

This demonstrated that the SSH tunnel and HAProxy/Horizon backend were
functioning.

The installation was subsequently confirmed to be working.

------------------------------------------------------------------------

## 11. Verify HAProxy and Horizon listeners

If Horizon access becomes problematic, check:

``` bash
sudo ss -lntp | grep -E ':80|:443'
```

The working configuration showed:

``` text
192.168.65.250:80
192.168.65.250:8000
192.168.65.250:8004
```

with HAProxy listening.

Check the VIP:

``` bash
ip addr show enp0s1
```

The successful configuration showed:

``` text
inet 192.168.65.2/24
inet 192.168.65.250/32
```

------------------------------------------------------------------------

## 12. Deployment verification checklist

After a successful deployment:

### Docker

``` bash
docker ps --format 'table {{.Names}}\t{{.Status}}' | sort
```

### Toolbox

``` bash
docker ps --filter name=kolla_toolbox
```

Expected:

``` text
kolla_toolbox   Up ... 
```

### Horizon

``` bash
curl -I http://192.168.65.250
```

Expected:

``` text
HTTP/1.1 302 Found
```

### Key services

Check that these are running/healthy:

``` text
keystone
glance_api
nova_api
nova_compute
nova_scheduler
nova_conductor
nova_libvirt
neutron_server
neutron_dhcp_agent
neutron_l3_agent
neutron_openvswitch_agent
cinder_api
cinder_scheduler
cinder_volume
placement_api
horizon
mariadb
rabbitmq
haproxy
openvswitch_db
openvswitch_vswitchd
kolla_toolbox
```

## Troubleshooting principle

For this lab, troubleshoot in this order:

``` text
VM / Linux
    |
    v
Network connectivity
    |
    v
Docker daemon
    |
    v
Kolla images
    |
    v
Kolla bootstrap
    |
    v
Kolla deployment
    |
    v
Container health
    |
    v
VIP / HAProxy
    |
    v
Horizon
    |
    v
OpenStack functional tests
```

Do not manually create Kolla-managed containers unless there is a
specific reason. Prefer correcting the Kolla configuration/state and
rerunning the appropriate Kolla-Ansible operation.
