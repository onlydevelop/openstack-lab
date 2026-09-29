# OpenStack Experimental Lab

> **Experimental / Learning Environment — Not for Production Use**

This repository documents an **experimental OpenStack lab environment** built for learning, experimentation, architecture exploration, troubleshooting, and hands-on development.

The setup is intentionally optimized for a **single-node, resource-constrained lab environment** running OpenStack inside an Ubuntu ARM64 virtual machine on an Apple Silicon Mac. It is **not a production reference architecture**.

## ⚠️ Important Disclaimer

This lab configuration is provided **for educational and experimental purposes only**.

It is **not intended, designed, tested, or warranted for production use**.

Do not use this configuration as-is for:

- Production cloud infrastructure
- Business-critical workloads
- Customer-facing services
- Sensitive or regulated workloads
- Financial or transactional systems
- High-availability environments
- Disaster-recovery environments
- Security-sensitive environments
- Systems requiring guaranteed performance or uptime
- Environments subject to regulatory, contractual, or compliance requirements

Any use outside a controlled laboratory environment is entirely at the user's own risk.

## What This Lab Is For

The primary objectives are to learn and experiment with OpenStack architecture, Kolla-Ansible, Nova, Neutron, Cinder, Glance, Keystone, Horizon, Placement, Heat, Swift, RabbitMQ, MariaDB, Open vSwitch, HAProxy, containerized OpenStack services, networking, block storage, troubleshooting, and automation.

## Lab Architecture

```text
                    MacBook Air M4
                     24 GB RAM
                         |
                         v
                        UTM
                         |
                         v
              Ubuntu Server 24.04 ARM64
                         |
                         v
                   Kolla-Ansible
                         |
              +----------+----------+
              |                     |
              v                     v
        Docker containers      Cinder LVM
              |                40 GB virtual disk
              |
     +--------+--------+
     | OpenStack       |
     | services        |
     +-----------------+
```

## Non-Production Characteristics

### Single-node design

The environment runs OpenStack services on a single virtual machine and does not provide production-grade control-plane, compute, network, or storage redundancy.

### Limited resources

The VM has substantially fewer resources than a typical production OpenStack environment. Resource contention can occur between OpenStack services, databases, messaging, storage, compute workloads, and the guest OS.

Performance observed in this lab must **not** be interpreted as representative of production OpenStack performance.

### Virtualized environment

OpenStack runs inside a VM, adding another virtualization layer between OpenStack and the physical Mac hardware. This can affect CPU, memory, disk I/O, networking, nested virtualization, and failure behavior.

### ARM64 platform

The lab uses ARM64 OpenStack container images because the host is Apple Silicon. Availability and maturity of individual components on ARM64 can differ from x86-64 environments. This is an experimental ARM64 environment, not a production compatibility statement.

## Availability and Reliability

This environment does **not** provide production-grade high availability.

A failure of the Mac, UTM, Ubuntu VM, virtual disk, network, or host OS can make the entire OpenStack environment unavailable.

Do not interpret this setup as providing fault tolerance, high availability, automatic disaster recovery, guaranteed service continuity, or guaranteed data durability.

## Data Loss Warning

**Do not store irreplaceable data in this OpenStack environment.**

Do not use the Cinder LVM backend as a substitute for production storage, and do not assume VM snapshots or virtual-disk backups constitute disaster recovery.

Important data should exist outside the lab in an independently managed backup system.

## Security Disclaimer

This configuration is **not a production security baseline**.

A production environment may require network segmentation, firewall policies, TLS, secure secret management, identity integration, access-control policies, credential rotation, hardened operating systems, container security, vulnerability management, audit logging, centralized monitoring, intrusion detection, backup/recovery controls, and compliance controls.

Lab passwords, SSH keys, API credentials, and other secrets must not be reused in production.

## Version and Configuration Disclaimer

The commands and configuration reflect a particular experimental environment and may depend on the OpenStack release, Kolla-Ansible version, Ubuntu release, ARM64 support, Docker version, UTM version, kernel version, network configuration, and hardware capabilities.

Future versions may change command syntax, defaults, image availability, configuration parameters, or supported workflows.

**Do not blindly apply these instructions to another environment.**

Validate the official documentation for the versions being deployed.

## No Warranty

This documentation is provided **"as is"**, without warranties or guarantees of any kind, express or implied.

No guarantee is made regarding correctness, completeness, availability, security, performance, compatibility, reliability, data preservation, or fitness for a particular purpose.

The person executing the commands is responsible for evaluating them before use.

## External Dependencies

The lab depends on software and projects maintained by third parties, including OpenStack, Kolla-Ansible, Ubuntu, UTM, Docker, Open vSwitch, RabbitMQ, MariaDB, and other open-source components.

Their respective licenses, documentation, support policies, and terms apply independently. This repository does not replace official project documentation.

## Production Deployment Guidance

A production OpenStack deployment should be designed separately for its actual requirements.

Important areas include:

```text
                    Production Design
                           |
        +------------------+------------------+
        |                  |                  |
     Compute            Network            Storage
        |                  |                  |
   HA / capacity      segmentation       redundancy
   scheduling         security           replication
        |                  |                  |
        +------------------+------------------+
                           |
                     Control Plane
                           |
             +-------------+-------------+
             |             |             |
          Identity      Database      Messaging
             |             |             |
             +-------------+-------------+
                           |
                    Observability
                           |
             +-------------+-------------+
             |             |             |
          Metrics        Logs          Traces
                           |
                     Backup / DR
```

A production architecture should address capacity planning, failure domains, hardware and network redundancy, storage redundancy, control-plane HA, database and messaging HA, load balancing, security, TLS, identity integration, monitoring, logging, backup/restoration, disaster recovery, upgrades, patching, operations, incident response, and compliance.

These concerns are intentionally outside the scope of this lab.

## Recommended Use

Use this lab to:

1. Learn OpenStack concepts.
2. Experiment with service configuration.
3. Create and destroy test workloads.
4. Study networking and storage behavior.
5. Learn Kolla-Ansible deployment workflows.
6. Practice troubleshooting.
7. Build proof-of-concepts.
8. Test automation and infrastructure-as-code ideas.
9. Develop familiarity before designing a properly engineered deployment.

## Before Using This Configuration Elsewhere

Review the relevant OpenStack release documentation, Kolla-Ansible documentation, platform architecture/support information, Ubuntu documentation, Docker documentation, storage and network requirements, security requirements, and backup/disaster-recovery requirements.

For production, create a **new architecture and configuration appropriate to the target environment** rather than treating this lab as a production template.

## Final Statement

**This project is an experimental OpenStack learning lab.**

Its purpose is to provide a practical environment for understanding OpenStack and related technologies.

**It should not be considered a production-ready OpenStack architecture, deployment guide, security baseline, performance benchmark, or operational runbook.**
