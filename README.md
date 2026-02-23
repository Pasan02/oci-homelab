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
I am currently in the process of creating the Virtual Machine (VM) on Oracle Cloud. This process has introduced a new, critical technical step: OCI does not natively support Debian 12, the chosen lightweight OS. Therefore, I am currently uploading a custom Debian 12 image to OCI Object Storage. This preparatory work ensures the new architecture runs on the desired minimalist OS for resource efficiency, guaranteeing 24/7 reliability and availability while adhering strictly to the Always Free Tier resources.

| Component | Technology / Status | Purpose |
| :--- | :--- | :--- |
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