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
│       │ Ansible  │ │Vaultwarden│ │          │                │
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

Proxmox VE is the central virtualization platform for the actual lab servers.

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

**Status: Implemented**

Debian 13 minimal server.

Responsibilities:

* self-hosted Git platform
* Gitea
* Gitea Actions
* Gitea Runner
* container registry
* Ansible controller

Address:

```text
192.168.122.10
```

Current services:

```text
GIT01
├── Docker Engine
├── Gitea
│   ├── Web UI
│   ├── Git repositories
│   ├── User authentication
│   ├── SSH Git access
│   └── Container Registry
├── Gitea Runner
└── Ansible
```

Gitea Web UI:

```text
http://192.168.122.10:3000
```

Git SSH:

```text
ssh://git@192.168.122.10:2222
```

The Gitea Runner executes CI/CD jobs using Docker containers.

The built-in Gitea Container Registry is used to store container images.

### APP01

**Status: Implemented**

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

Current services:

```text
APP01
├── Docker Engine
└── Vaultwarden
    ├── Docker Compose
    ├── HTTPS
    └── Persistent data
```

Vaultwarden is deployed and managed through Ansible.

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
       │ Gitea Actions
       ▼
┌─────────────┐
│ Gitea Runner│
│    GIT01    │
└──────┬──────┘
       │
       │ Ansible
       ▼
┌─────────────┐
│    APP01    │
│    Docker   │
└──────┬──────┘
       │
       ▼
  Vaultwarden
```

This creates a self-hosted development and deployment workflow without relying on GitHub, GitLab, or a cloud CI platform.

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

The Git workflow has been tested with:

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

Gitea Actions is used for CI/CD automation.

The Gitea Runner is registered on GIT01 and executes jobs using Docker.

## Container Registry

**Status: Implemented**

Gitea's built-in OCI-compatible Container Registry is used instead of deploying a separate registry service.

The registry is available through:

```text
192.168.122.10:3000
```

Container images can be built, pushed, pulled, and executed through the registry.

Example:

```text
Docker build
     │
     ▼
Gitea Runner
     │
     ▼
Gitea Container Registry
     │
     ├── push
     │
     └── pull
```

Because the lab registry uses HTTP rather than HTTPS, the Docker daemon on GIT01 is configured to explicitly allow the lab registry as an insecure registry.

## Ansible

**Status: Implemented**

GIT01 acts as the Ansible controller.

Ansible is stored in the same Git repository as the infrastructure configuration:

```text
infrastructure-lab/
├── ansible.cfg
└── ansible/
    ├── inventory/
    ├── playbooks/
    └── roles/
```

Current playbooks:

```text
ansible/
├── inventory/
│   └── hosts.yml
├── playbooks/
│   ├── app-baseline.yml
│   └── vaultwarden.yml
└── roles/
    ├── users/
    ├── ssh/
    ├── packages/
    ├── docker/
    └── vaultwarden/
```

The baseline configuration manages:

* service users
* SSH configuration
* baseline packages
* Docker repository and packages
* Docker service
* Docker group membership

The Vaultwarden role manages:

* application directory
* TLS certificate and key
* application environment
* Docker Compose configuration
* Vaultwarden deployment

The baseline playbook has been verified for idempotency:

```text
First run  → changes
Second run → 0 changes
```

A controlled drift test was also performed by stopping Docker manually and allowing Ansible to restore the desired state.

## Application Layer

**Status: Implemented — Vaultwarden**

APP01 hosts application workloads using Docker.

The first application is Vaultwarden:

```text
APP01
│
└── Docker
    │
    └── Vaultwarden
         ├── HTTPS
         └── persistent data
```

Vaultwarden is deployed using:

* Docker Compose
* persistent Docker volume
* Ansible configuration management
* built-in Rocket TLS
* self-signed lab certificate
* configuration stored outside the Git repository where appropriate

Vaultwarden is accessible at:

```text
https://192.168.122.20:8443
```

The application has been tested for:

* container startup
* HTTPS access
* user registration and login
* persistent data
* container recreation
* Docker Compose deployment

Secrets and private keys are not committed to Git.

## CI/CD

**Status: Implemented — deployment pipeline**

The current CI/CD workflow is:

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
    ├── Validate app-baseline.yml
    ├── Validate vaultwarden.yml
    │
    ▼
    ├── Run app-baseline.yml
    │
    ▼
    └── Run vaultwarden.yml
            │
            ▼
          APP01
            │
            ▼
          Docker
            │
            ▼
       Vaultwarden
```

The pipeline is stored in:

```text
.gitea/workflows/deploy.yaml
```

The deployment pipeline performs two stages.

### Validation

Both Ansible playbooks are syntax-checked before deployment:

```text
app-baseline.yml
        │
        ├── syntax-check
        │
        ▼
vaultwarden.yml
        │
        └── syntax-check
```

The deployment job depends on successful validation.

```yaml
needs: validate
```

Therefore a validation failure prevents the deployment job from running.

### Deployment

After successful validation, the pipeline:

1. checks out the repository
2. prepares the Ansible SSH key from Gitea Secrets
3. runs the APP01 baseline playbook
4. runs the Vaultwarden deployment playbook

The CI environment uses a dedicated container image containing:

* Ansible
* OpenSSH client
* Git
* CA certificates

The repository is the source of truth for the deployment configuration.

A new server still requires minimal bootstrap access such as networking, SSH, and initial credentials before Ansible can take over the configuration.

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
│ Phase 6  Runner + Registry                [✓] │
│ Phase 7  Ansible baseline                 [✓] │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ APPLICATION                                 │
├─────────────────────────────────────────────┤
│ Phase 8  Docker APP01                     [✓] │
│ Phase 9  Vaultwarden                      [✓] │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ DELIVERY                                    │
├─────────────────────────────────────────────┤
│ Phase 10 CI/CD                            [~] │
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

`[~]` indicates that the main CI/CD deployment workflow is implemented, with final idempotency and failure-path verification still remaining.

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
* Gitea user and repository created
* SSH Git authentication verified
* Gitea Actions Runner deployed and registered
* Gitea Actions workflow execution verified
* Gitea Container Registry configured
* Container image push/pull verified
* Ansible controller configured on GIT01
* APP01 baseline automated with Ansible
* Ansible idempotency verified
* Docker installed and managed through Ansible
* Docker application platform verified on APP01
* Vaultwarden deployed with Docker Compose
* Vaultwarden HTTPS configured
* Vaultwarden persistence verified
* CI image created for Ansible deployment
* Gitea Actions validation stage implemented
* Automated Ansible deployment to APP01 implemented
* CI/CD deployment successfully tested end-to-end

### In Progress / Next

* Verify CI/CD idempotency
* Verify CI/CD failure gate
* ZBX01
* Zabbix monitoring
* Incident/RCA scenarios
* Final documentation

## Repository Structure

The repository reflects the actual infrastructure implementation:

```text
infrastructure-lab/
├── .gitea/
│   └── workflows/
│       └── deploy.yaml
│
├── ansible/
│   ├── inventory/
│   │   └── hosts.yml
│   ├── playbooks/
│   │   ├── app-baseline.yml
│   │   └── vaultwarden.yml
│   └── roles/
│       ├── users/
│       ├── ssh/
│       ├── packages/
│       ├── docker/
│       └── vaultwarden/
│
├── ci/
│   └── Dockerfile
│
├── ansible.cfg
├── Dockerfile
└── README.md
```

The repository structure evolves together with the infrastructure rather than documenting components that do not yet exist.

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

Manual configuration is used when appropriate, but recurring configuration progressively moves into:

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
* build self-hosted CI/CD pipelines
* deploy applications through automation
* manage container images through a registry
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
5. Show Ansible
       ↓
6. Push a change
       ↓
7. Show CI/CD pipeline
       ↓
8. Show automated deployment on APP01
       ↓
9. Show Vaultwarden
       ↓
10. Show Zabbix monitoring
       ↓
11. Introduce a controlled failure
       ↓
12. Troubleshoot it
       ↓
13. Show RCA
```

The objective is to demonstrate not only that the services work, but that the infrastructure can be **operated, automated, monitored, troubleshot, and explained**.

## Lab vs Production Experience

This project represents practical home-lab experience.

Technologies implemented here should not be presented as commercial production experience unless separately supported by professional work experience.

The value of the project is demonstrating hands-on understanding of infrastructure concepts and the ability to build and operate a small environment end-to-end.
