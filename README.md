# Infrastructure Engineering Lab

A hands-on infrastructure and DevOps home lab designed to simulate a small on-premise engineering environment.

The project focuses on practical infrastructure engineering: virtualization, Linux administration, networking, self-hosted Git, containerization, CI/CD, configuration management, monitoring, and incident troubleshooting.

The lab is built incrementally on a single Arch Linux host using KVM/libvirt and Proxmox VE as the virtualization layer.

> This is a personal home lab created for practical learning and interview preparation. It is not a production environment.

## Goals

The main goals of the project are to practice and demonstrate:

* Linux server administration
* KVM/libvirt and Proxmox virtualization
* IP networking and troubleshooting
* Docker and container management
* Self-hosted Git infrastructure
* Gitea Actions and CI/CD
* Ansible configuration management
* Container registries
* Monitoring with Zabbix
* Infrastructure troubleshooting and RCA
* Reproducible infrastructure configuration
* Operational thinking and documentation

The project is intentionally kept small enough to run on a personal workstation while still representing a realistic infrastructure workflow.

## Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                         ARCH HOST                            │
│                                                             │
│  Arch Linux                                                 │
│  KVM / QEMU / libvirt                                       │
│  qemu:///system                                              │
│                                                             │
│  libvirt default network: 192.168.122.0/24                  │
│                                                             │
│                         │                                   │
│                         ▼                                   │
│                  ┌───────────────┐                          │
│                  │   PROXMOX01   │                          │
│                  │ 192.168.122.2 │                          │
│                  │               │                          │
│                  │ Proxmox VE     │                          │
│                  │ vmbr0          │                          │
│                  └───────┬───────┘                          │
│                          │                                  │
│             ┌────────────┼────────────┐                     │
│             │            │            │                     │
│             ▼            ▼            ▼                     │
│       ┌──────────┐ ┌──────────┐ ┌──────────┐                │
│       │  GIT01   │ │  APP01   │ │  ZBX01   │                │
│       │ .122.10  │ │ .122.20  │ │ .122.30  │                │
│       │          │ │          │ │          │                │
│       │ Gitea    │ │ Docker   │ │ Zabbix   │                │
│       │ Runner   │ │ Apps     │ │ Server   │                │
│       │ Registry │ │          │ │          │                │
│       └──────────┘ └──────────┘ └──────────┘                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

All lab VMs use the same `vmbr0` network.

### Network

| System          |          Address | Purpose                 |
| --------------- | ---------------: | ----------------------- |
| Libvirt gateway |  `192.168.122.1` | NAT gateway             |
| PROXMOX01       |  `192.168.122.2` | Virtualization platform |
| GIT01           | `192.168.122.10` | Git / CI infrastructure |
| APP01           | `192.168.122.20` | Application server      |
| ZBX01           | `192.168.122.30` | Monitoring server       |

The lab intentionally uses a single network segment rather than introducing unnecessary routing or VLAN complexity.

## Virtualization

The physical lab host is an Arch Linux workstation.

KVM/QEMU and libvirt provide the first virtualization layer:

```text
Arch Linux
    │
    └── KVM / QEMU / libvirt
            │
            └── PROXMOX01
                    │
                    ├── GIT01
                    ├── APP01
                    └── ZBX01
```

Proxmox VE is therefore the central virtualization platform for the actual lab servers.

The Proxmox VM uses:

* ~7 vCPU
* ~7.75 GB RAM
* 40 GB system disk
* 100 GB VM storage disk
* nested KVM

The Proxmox VM storage is provided by an LVM-thin storage named `vmdata`.

## Servers

### PROXMOX01

**Status: Implemented**

Proxmox VE runs as a nested virtualization host inside KVM/libvirt.

Responsibilities:

* VM lifecycle management
* VM storage
* virtual networking
* nested virtualization

Address:

```text
192.168.122.2
```

### GIT01

**Status: Implemented — base server + Gitea**

Debian 13 minimal server.

Responsibilities:

* self-hosted Git platform
* Gitea
* Git repositories
* future Gitea Actions runner
* future container registry workflow

Address:

```text
192.168.122.10
```

Current services:

```text
GIT01
├── Docker Engine
└── Gitea
    ├── Web UI
    ├── Git repositories
    ├── User authentication
    └── SSH Git access
```

Gitea Web UI:

```text
http://192.168.122.10:3000
```

Git SSH:

```text
ssh://git@192.168.122.10:2222
```

Gitea is deployed using Docker Compose with persistent Docker storage.

### APP01

**Status: Implemented — base server**

Debian 13 minimal server.

Responsibilities:

* application workloads
* Docker runtime
* persistent application services
* CI/CD deployment target

Address:

```text
192.168.122.20
```

The application layer will be built on top of this server.

### ZBX01

**Status: Planned**

Debian 13 minimal server dedicated to monitoring.

Responsibilities:

* Zabbix Server
* monitoring backend
* monitoring Web UI
* infrastructure monitoring

Address:

```text
192.168.122.30
```

## DevOps Platform

The DevOps platform is built around Gitea.

```text
Developer
    │
    │ git push
    ▼
┌─────────────┐
│    Gitea    │
│    GIT01    │
└──────┬──────┘
       │
       │ Actions
       ▼
┌─────────────┐
│ Gitea Runner│
│    GIT01    │
└──────┬──────┘
       │
       │ build / test / deploy
       ▼
┌─────────────┐
│    APP01    │
│    Docker   │
└─────────────┘
```

This creates a complete self-hosted development and deployment workflow without relying on GitHub, GitLab, or a cloud CI platform.

## Gitea

**Status: Implemented**

Gitea is deployed on GIT01 using Docker Compose.

Current repository:

```text
jaj/infrastructure-lab
```

Git access is provided through SSH:

```text
ssh://git@192.168.122.10:2222/jaj/infrastructure-lab.git
```

The Git workflow has been tested with SSH authentication.

The intended workflow is:

```text
clone
  │
  ▼
edit
  │
  ▼
commit
  │
  ▼
push
  │
  ▼
Gitea
  │
  ▼
pull
```

## Containerization

Docker is used in two different roles.

### GIT01

Docker provides the infrastructure runtime for:

* Gitea
* Gitea Runner
* CI-related tooling

### APP01

Docker provides the application runtime for:

* application services
* Vaultwarden
* future deployed workloads

This separation keeps the Git/CI infrastructure independent from the application host.

## Ansible

**Status: Planned**

GIT01 will act as the Ansible controller.

The goal is to configure APP01 reproducibly instead of performing all configuration manually.

Planned structure:

```text
GIT01
└── Ansible
    ├── inventory
    ├── playbooks
    └── roles
            │
            ▼
          APP01
```

Ansible will manage tasks such as:

* user configuration
* SSH configuration
* package installation
* system configuration
* Docker prerequisites
* application prerequisites

A key verification point will be idempotency:

```text
First run  → changes
Second run → 0 changes
```

## Application Layer

**Status: Planned**

APP01 will host the application workloads using Docker.

The first major service will be Vaultwarden.

```text
APP01
│
└── Docker
    │
    └── Vaultwarden
         │
         └── persistent data
```

The deployment will use:

* Docker Compose
* persistent volumes
* configuration separated from secrets
* controlled restart behavior
* reproducible deployment

Secrets will not be committed to Git.

## CI/CD

**Status: Planned**

The target delivery workflow is:

```text
Developer
    │
    │ git push
    ▼
Gitea
    │
    │ Gitea Actions
    ▼
Gitea Runner
    │
    ├── validate
    ├── test
    ├── build
    └── deploy
            │
            ▼
          APP01
            │
            ▼
          Docker
            │
            ▼
       Application
```

The pipeline should demonstrate both successful and failed deployments.

A successful change should result in:

```text
commit
  → push
  → pipeline
  → validation
  → build
  → deployment
  → running application
```

A deliberately broken change should fail during the pipeline and prevent deployment.

## Monitoring

**Status: Planned**

ZBX01 will provide centralized monitoring through Zabbix.

Initial monitoring targets:

```text
                 ┌──────────────┐
                 │    ZBX01     │
                 │    Zabbix    │
                 └──────┬───────┘
                        │
             ┌──────────┼──────────┐
             │          │          │
             ▼          ▼          ▼
           GIT01      APP01      Proxmox
```

Metrics will include:

* CPU
* memory
* disk usage
* network availability
* service availability
* SSH
* Docker
* Gitea
* Vaultwarden

Triggers will be configured for meaningful infrastructure failures.

## Incident Response and RCA

**Status: Planned**

The lab includes controlled failures to practice real operational troubleshooting.

The troubleshooting methodology is:

```text
Observe
   ↓
Form hypothesis
   ↓
Run checks
   ↓
Collect evidence
   ↓
Identify root cause
   ↓
Apply fix
   ↓
Verify recovery
   ↓
Document RCA
```

Planned incidents include:

* application unavailable
* failed CI/CD deployment
* service failure
* networking failure

The goal is not simply to fix the issue, but to demonstrate a structured troubleshooting process.

## Project Phases

```text
┌─────────────────────────────────────────────┐
│ FOUNDATION                                  │
├─────────────────────────────────────────────┤
│ Phase 0  Host preparation             [✓]   │
│ Phase 1  PROXMOX01                     [✓]   │
│ Phase 2  Proxmox network               [✓]   │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ SERVERS                                     │
├─────────────────────────────────────────────┤
│ Phase 3  GIT01                           [✓] │
│ Phase 4  APP01                           [✓] │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ DEVOPS PLATFORM                             │
├─────────────────────────────────────────────┤
│ Phase 5  Gitea                           [✓] │
│ Phase 6  Runner + Registry                [ ] │
│ Phase 7  Ansible baseline                 [ ] │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ APPLICATION                                 │
├─────────────────────────────────────────────┤
│ Phase 8  Docker APP01                     [ ] │
│ Phase 9  Vaultwarden                      [ ] │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ DELIVERY                                    │
├─────────────────────────────────────────────┤
│ Phase 10 CI/CD                            [ ] │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ MONITORING                                  │
├─────────────────────────────────────────────┤
│ Phase 11 ZBX01                            [ ] │
│ Phase 12 Zabbix monitoring                [ ] │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ OPERATIONS                                  │
├─────────────────────────────────────────────┤
│ Phase 13 Incident / RCA                   [ ] │
│ Phase 14 Documentation                    [ ] │
└─────────────────────────────────────────────┘
```

## Current Status

### Completed

* Arch Linux host prepared for virtualization
* KVM/QEMU/libvirt configured
* Proxmox VE deployed as nested virtualization host
* Proxmox VM storage configured
* Proxmox networking configured
* Nested KVM verified
* GIT01 deployed
* APP01 deployed
* Debian 13 minimal servers configured
* Static server addressing configured
* SSH access verified
* Internet connectivity verified
* GIT01 ↔ APP01 connectivity verified
* Docker installed on GIT01
* Gitea deployed using Docker Compose
* Gitea Web UI configured
* Gitea user created
* Gitea repository created
* SSH authentication from Arch host verified

### In Progress / Next

* Gitea Runner
* Container Registry workflow
* Ansible baseline
* Docker application platform on APP01
* Vaultwarden
* CI/CD
* Zabbix
* Incident/RCA scenarios
* Final documentation

## Repository Structure

The repository will evolve together with the infrastructure.

A target structure is:

```text
infrastructure-lab/
├── README.md
│
├── ansible/
│   ├── inventory/
│   ├── playbooks/
│   └── roles/
│
├── gitea/
│   └── compose.yaml
│
├── docker/
│   └── ...
│
├── applications/
│   └── vaultwarden/
│       ├── compose.yaml
│       └── ...
│
├── monitoring/
│   └── zabbix/
│
├── docs/
│   ├── architecture.md
│   ├── networking.md
│   ├── operations.md
│   └── incidents/
│
└── .gitignore
```

The repository structure will reflect the actual implementation rather than documenting components that do not yet exist.

## Design Principles

### Implementation over theory

Every major component is implemented, tested, and documented.

The workflow is:

```text
Analyze
   ↓
Design
   ↓
Implement
   ↓
Verify
   ↓
Troubleshoot
   ↓
Document
   ↓
Review
```

### Small and realistic

The lab intentionally avoids unnecessary complexity.

It does not attempt to simulate a large enterprise environment with dozens of VMs, Kubernetes clusters, complex HA configurations, or multiple redundant services.

The objective is to demonstrate infrastructure engineering fundamentals clearly.

### Reproducibility

Infrastructure configuration should be reproducible wherever practical.

Manual configuration is used when appropriate, but recurring configuration should progressively move into:

* Git
* Ansible
* Docker Compose
* CI/CD

### Troubleshooting first

Failures are treated as part of the project rather than something to avoid.

Each controlled incident should result in evidence, a root cause, a fix, and verification.

### Security basics

The lab follows practical security principles appropriate for its scope:

* SSH key authentication
* least-privilege access where practical
* secrets excluded from Git
* controlled service exposure
* separate infrastructure and application responsibilities

## Technologies

### Infrastructure

* Arch Linux
* Debian 13
* KVM/QEMU
* libvirt
* Proxmox VE
* Linux networking

### DevOps

* Git
* Gitea
* Gitea Actions
* Gitea Runner
* Docker
* Docker Compose
* Ansible

### Monitoring

* Zabbix

### Application

* Vaultwarden
* Docker-based services

## What This Project Demonstrates

The lab is intended to demonstrate practical ability to:

* build a Linux-based infrastructure from scratch
* operate virtualized infrastructure
* configure and troubleshoot networking
* administer Debian Linux servers
* deploy and operate self-hosted infrastructure services
* work with Docker and Docker Compose
* manage Git infrastructure
* automate server configuration with Ansible
* build CI/CD pipelines
* deploy applications through automation
* monitor infrastructure
* investigate infrastructure failures
* perform root-cause analysis
* document infrastructure clearly

## Interview Demo

A complete demonstration of the project should follow a practical path:

```text
1. Show architecture
       ↓
2. Show Proxmox
       ↓
3. Show GIT01 / APP01 / ZBX01
       ↓
4. Show Gitea
       ↓
5. Push a change
       ↓
6. Show CI/CD pipeline
       ↓
7. Show deployment on APP01
       ↓
8. Show Zabbix monitoring
       ↓
9. Introduce a controlled failure
       ↓
10. Troubleshoot it
       ↓
11. Show RCA
```

The objective is to demonstrate not only that the services work, but that the infrastructure can be **operated, automated, monitored, troubleshot, and explained**.

## Lab vs Production Experience

This project represents practical home-lab experience.

Technologies implemented here should not be presented as commercial production experience unless separately supported by professional work experience.

The value of the project is demonstrating hands-on understanding of infrastructure concepts and the ability to build and operate a small environment end-to-end.

