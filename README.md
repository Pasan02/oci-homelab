# From Laptop to Cloud: Self-Hosted Media &amp; Document Solution

## Introduction
In an era dominated by large, proprietary cloud services, I embarked on a project to build my own secure, cost-effective, and fully-controlled hosting solution for my personal photos and files. This journey was motivated by a desire for data ownership, a need to escape escalating subscription costs, and a commitment to utilizing and learning new technologies.

This repository documents the initial concept, unexpected technical hurdles, and the pivotal decision to transition to a reliable, Always Free Tier cloud architecture. It is designed for both a general technical audience and potential employers interested in the practical application of cloud, containerization, and system administration skills.

## The Core Problem: Data Sovereignty and Cost
Like many, I relied on solutions like Google Photos and Google Drive. While convenient, the long-term costs and the reliance on a third-party ecosystem felt restrictive. I needed a custom solution that offered:
*   **Total Control:** Full ownership and management of my data and the software running it.
*   **Accessibility:** Access to my files from any device, anywhere.
*   **Cost-Effectiveness:** A solution designed to utilize the Oracle Cloud Always Free Tier resources to eliminate recurring monthly fees.

To address my specific needs, I selected two powerful, lightweight open-source applications:
*   **<a href="https://immich.app/">Immich</a>:** A high-performance self-hosted photo and video backup solution.
*   **<a href="https://docs.paperless-ngx.com/">Paperless-ngx</a>:** A document management system designed to archive and organize scanned documents and digital files.

Crucially, **Docker Compose** will be used for the deployment of both services, leveraging its efficiency for managing multi-container applications.

## Phase 1: The Home Server Concept and the Wake-on-LAN Hurdle
My initial plan was to repurpose an old laptop as a dedicated home server running Debian, chosen for its lightweight footprint and stability.

The architecture was designed for low power consumption:
*   **OS:** Debian (Lightweight server installation).
*   **Access:** Cloudflared tunnel for secure remote management.
*   **Power Management:** Wake-on-LAN (WoL) and scripts to manage uptime.

### The Unforeseen Challenge
Setting up WoL was successful, but a critical hardware limitation emerged: the laptop lacked the BIOS-level feature for automatic power-on after a power outage. This single flaw—rendering the server inaccessible after a simple power flicker—defeated the entire goal of "access from anywhere."

## Phase 2: Pivoting to Oracle Cloud Infrastructure (OCI)
Faced with hardware constraints, I made the strategic decision to pivot to a virtualized cloud environment. The search for a suitable platform led me to **Oracle Cloud Infrastructure (OCI)**, specifically its **Always Free Tier**.

OCI's free tier provides perpetual compute instances and storage that allow me to meet the project's reliability and cost requirements without compromise.

## Phase 3: The Reality Check - Resource Constraints & Architecture Shift
My initial attempt at deploying the stack on OCI's **AMD Micro instance** (1 OCPU, 1 GB RAM) proved to be a significant bottleneck.

### The Failure Point
Upon deploying the full stack (Nginx + Immich + Database + Redis), the system immediately became unresponsive. The terminal lagged, and Nginx returned `502 Bad Gateway` errors.
*   **Root Cause:** Immich is a resource-intensive application, requiring a Machine Learning engine, Redis, and Postgres. The 1 GB RAM ceiling of the Micro instance was instantly hit, causing aggressive swapping and thrashing.

### The Solution: ARM Ampere Architecture
To resolve this, I migrated the entire infrastructure to OCI's **Always Free ARM Ampere** tier (`VM.Standard.A1.Flex`).
*   **Specs:** 4 OCPUs (ARM64), 24 GB RAM.
*   **Impact:** This massive increase in memory (24x) eliminated all performance bottlenecks, allowing Immich and Paperless-ngx to run smoothly side-by-side.

## Phase 4: The OS Dilemma - Debian vs. Ubuntu on ARM
While my preference (and the original plan) was to use **Debian** for its stability and minimal footprint, a platform-specific hurdle emerged during the ARM migration.

### The Hurdle
Oracle Cloud's web console has strict filtering rules for image compatibility. When selecting the ARM Ampere shape, the standard "Debian" platform image was often hidden or marked as incompatible in the easy-select dropdowns, as Oracle's default Debian images are primarily built for x86 (AMD/Intel) architectures.

### The Decision
Rather than fighting the platform by building and uploading a custom Debian ARM image (which introduces maintenance overhead), I opted to switch to **Ubuntu 24.04 LTS**.
*   **Reasoning:** Ubuntu has first-class, official support for ARM on OCI. It is readily available in the "Platform Images" list, ensuring seamless updates and compatibility with the `VM.Standard.A1.Flex` shape.
*   **Trade-off:** Minimal. Ubuntu is based on Debian, uses `apt`, and runs Docker identically, satisfying all project requirements with zero friction.

## Phase 5: Implementation Hurdles & Solutions

### 1. The Firewall (iptables) Trap
Default OCI images come with strict `iptables` rules. When attempting to open ports 80 (HTTP) and 443 (HTTPS), standard tutorials failed.
*   **Error:** `iptables: Index of insertion too big.`
*   **Cause:** The default firewall chain was empty or shorter than expected, so trying to insert a rule at index `#6` (a common copy-paste command) failed.
*   **Fix:** We simplified the command to insert rules at the *top* of the chain (`iptables -I INPUT 1 ...`) or simply append them, ensuring traffic was allowed.

### 2. The "Billable Size" Scare
During the creation of a custom backup image, OCI reported a "Billable Size: 2GB," raising concerns about accidental costs.
*   **Clarification:** OCI's Always Free Tier includes **10 GB** of Object Storage. The "Billable Size" simply refers to the space that *counts* against your quota, not necessarily a charged amount. Since 2GB < 10GB, the backup remained free.

### 3. Docker Image Manifests
The initial `docker-compose.yml` for Immich likely used specific SHA256 digests for Redis and Postgres that were either outdated or x86-specific.
*   **Fix:** We switched to standard version tags (e.g., `redis:6.2-alpine`, `postgres:14-alpine`) in the Docker Compose file. Docker's multi-arch support then automatically pulled the correct `arm64` images for the new Ampere instance.

## Current Status and Technical Architecture
I have successfully migrated to the high-performance ARM tier and established the foundational application stack.

| Component | Technology / Status | Purpose |
| :--- | :--- | :--- |
| **Network** | **VCN, Subnets, Gateway** | Configured for secure public and private traffic. |
| **Security** | **Security Lists & Internal Firewall** | Ingress rules set for web traffic (80/443); SSH restricted. |
| **Operating System** | **Ubuntu 24.04 (ARM64)** | High-performance OS fully supported on OCI Ampere. |
| **Application Stack** | **Immich** | Core photo and video management service running in Docker. |
| **Reverse Proxy** | **Nginx** | Handles traffic routing and load balancing. |
| **Compute** | **OCI Ampere (4 OCPU, 24GB RAM)** | Provides substantial resources for AI/ML workloads. |

## Key Decisions Remaining
Before proceeding with the deployment of Paperless-ngx, I must finalize the strategy for the critical components:
1.  **Public Access Method:** Currently relying on Nginx Reverse Proxy with standard HTTP. SSL certs (LetsEncrypt) and domain configuration are pending.
2.  **Backup Plan:** A robust, automated backup solution is required that aligns with the OCI Always Free Tier.
3.  **VPN vs. Bastion:** While the Bastion is secure, I am considering setting up a VPN (WireGuard) to allow direct access to the server via a fixed internal IP.).
2.  **Backup Plan:** A robust, automated backup solution is required that aligns with the OCI Always Free Tier to avoid costs.
3.  **VPN vs. Bastion:** While the Bastion is secure, it introduces friction for daily management. I am considering setting up a VPN (like WireGuard or OpenVPN) to allow direct access to the server via a fixed internal IP, simplifying the workflow.

## Troubleshooting Log: The Silent Bastion Incident

### The Issue
During the configuration of the OCI Bastion service, a perplexing connectivity issue arose. Despite the Bastion tunnel establishing successfully (authentication passed), any attempt to SSH into the target VM through the tunnel resulted in the error: `Connection closed by ::1 port 9000`.

This behavior was confusing because:
1.  **Direct SSH worked:** I could SSH directly to the VM using its public IP (temporarily allowed).
2.  **Bastion Auth worked:** The verbose SSH logs (`ssh -vvv`) showed the tunnel handshake completing.
3.  **VM Logs were silent:** `sudo journalctl -u ssh` on the VM showed *zero* connection attempts from the Bastion's private IP.

### The Investigation
We systematically ruled out potential causes:
-   **Client-Side:** Forced IPv4 usage (`127.0.0.1` vs `localhost`) to rule out Windows IPv6 tunneling quirks.
-   **VM Firewall:** Verified localized firewalls (`iptables`, `nftables`) were inactive or allowing traffic.
-   **OCI Ingress:** Confirmed the Security List allowed TCP traffic on port 22 from the Bastion's private endpoint IP.
-   **Network Traffic:** Running `tcpdump -i any host [Bastion_IP]` on the VM revealed that **no packets** were arriving from the Bastion.

### The Root Cause: Missing Egress Rules
The culprit was identified in the **OCI Security List Egress Rules**. 

The Bastion service injects a hidden VNIC into the target subnet to forward traffic. While I had correctly configured **Ingress Rules** to allow traffic *into* the subnet, I had inadvertently deleted the default **Egress Rule** (0.0.0.0/0). 

Because OCI Security Lists are stateful but require explicit permission to initiate outbound traffic, the Bastion could receive my connection request but was blocked by the subnet rules from sending that traffic *out* to the VM (even though they are in the same subnet).

### The Fix & Lesson
**The Fix:** Added a stateful Egress Rule to the subnet Security List allowing `0.0.0.0/0` (All Protocols).

**The Lesson:** 
1.  **Bastions live in the subnet:** They are subject to the same Security List rules as the VMs they manage.
2.  **Ingress != Connectivity:** Allowing traffic *in* isn't enough; the service must be allowed to send traffic *out* to its target.
3.  **Tcpdump doesn't lie:** When logs are silent, packet captures are the definitive source of truth for network path issues.