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

## Current Status and Technical Architecture
I have successfully established the foundational networking infrastructure within Oracle Cloud. Key components, including the **Virtual Cloud Network (VCN)**, **Internet Gateway**, **Route Tables**, and a **Public Subnet**, are now operational. Security lists have been configured to allow HTTPS and HTTP traffic, while restricting management access.

Crucially, I have deployed and tested a **Bastion Host**. This ensures secure, tunnelled SSH access to my private resources without exposing them directly to the public internet.

Simultaneously, I am preparing the compute environment. Since OCI does not natively support Debian 12, I am uploading a custom Debian 12 image.

| Component | Technology / Status | Purpose |
| :--- | :--- | :--- |
| **Network** | **VCN, Subnets, Gateway** | configured for secure public and private traffic. |
| **Security** | **Security Lists & Bastion** | Ingress rules set for web traffic; SSH restricted via Bastion. |
| **Operating System** | Debian 12 (Custom Image) | Minimalist, stable OS for resource efficiency. |
| **Application Stack** | Immich &amp; Paperless-ngx | Core photo and file management services. |
| **Deployment** | Docker Compose | Used for consistent, reproducible, and efficient application deployment. |
| **SSH Access** | Bastion Host / Jump Server | Utilized for secure, controlled access to the VM, minimizing the attack surface. |
| **Public Access** | *(Undecided)* | Method for securely exposing Immich/Paperless-ngx web interfaces to the internet. |
| **Data Backup** | *(Undecided)* | Strategy for ensuring data redundancy and recoverability. |

## Key Decisions Remaining
Before proceeding with the deployment of services, I must finalize the strategy for the two most critical components:
1.  **Public Access Method:** How will the Immich and Paperless-ngx web interfaces be securely exposed to the public internet? (e.g., VPN, Cloudflare Tunnel, or a reverse proxy).
2.  **Backup Plan:** A robust, automated backup solution is required that aligns with the OCI Always Free Tier to avoid costs.

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