# Linux System Administration: 6-Week Challenge

## Overview
This repository documents a 6-week progressive mastery curriculum focused on Linux system administration on a Virtual Private Server (VPS). The program uses a "rebuild-to-learn" methodology. By intentionally wiping the VPS at the end of every week, this project treats infrastructure as disposable and reproducible code, building practical command-line skills and technical confidence.

## Curriculum Structure
The challenge follows a two-phase strategic approach, migrating from a Debian-based environment to an enterprise-grade Red Hat-style architecture.

### Phase 1: The Ubuntu Track (Debian-Based Administration)
This phase utilises Ubuntu Server to build a foundation in server navigation, cryptographic security, and web hosting.
* **Week 1 | Linux Fundamentals:** Gaining absolute control over the environment via SSH without a GUI. Tasks include creating nested directory structures, configuring non-root sudo users to enforce the Principle of Least Privilege (PoLP), and managing lifecycle updates.
* **Week 2 | Hardening and Perimeter Security:** Transitioning to cryptographic key pairs by generating local SSH keys and disabling root and password authentication via `/etc/ssh/sshd_config`. Perimeter defence is established by configuring UFW to strictly allow OpenSSH.
* **Week 3 | Web Architecture and Service Persistence:** Deploying a production-ready Node.js or Vue.js application using Nginx as a reverse proxy. The application lifecycle and process persistence are managed using standard `systemd` service files.

### Phase 2: The Rocky Linux Track (Enterprise RHEL-Based Administration)
This phase focuses on making your skills adaptable across different Linux systems by applying it to a strict, enterprise-level environment using Rocky Linux.
* **Week 4 | The Translation Layer:** Mapping Debian commands to RHEL equivalents, transitioning from the `apt` to the `dnf` package manager, and recreating user permissions in a new architectural layout.
* **Week 5 | Advanced Hardening:** Implementing zone-based network security utilising `firewalld` and navigating Mandatory Access Control (MAC) by keeping SELinux in "Enforcing" mode.
* **Week 6 | Full-Stack Enterprise Production:** Executing a complete, secure deployment. This final persistent build requires precise SELinux configuration, specifically applying the `httpd_can_network_connect` boolean to allow the Nginx proxy to route traffic to the backend application.

## Core Technologies & Tools
* **Operating Systems:** Ubuntu Server 24.04, Rocky Linux
* **Web & Services:** Nginx, Node.js, systemd
* **Security & Networking:** SSH Cryptography, UFW, firewalld, SELinux
* **Package Management:** apt, dnf