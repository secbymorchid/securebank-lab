# Phase 2: SecureBank Architecture & Trust Boundaries

This directory outlines the high-level system architecture, network segmentation, and security posture of the SecureBank infrastructure.

Before diving into infrastructure-as-code or server configuration, it is critical to establish a shared mental model of how data flows through the system. This document defines the structural blueprint of SecureBank, illustrating how we separate concerns, enforce least privilege, and defend against external threats.

## 1. Architectural Context

SecureBank is a modern, digital-first retail banking platform designed to process sensitive financial transactions while withstanding targeted cyberattacks. The platform serves three distinct user groups, each with fundamentally different access requirements:

- **Retail Clients:** Accessing the platform from public networks to view balances, initiate transfers, and manage cards.
- **Branch Employees:** Accessing the platform from corporate networks to manage agency operations, perform KYC validations, and support clients.
- **Security Operations Center (SOC) Analysts:** Accessing the system via dedicated secure terminals to monitor audit logs, investigate anomalies, and respond to active incidents.

**Scope exclusions.** The following are explicitly out of scope for this project, to keep the threat model and hardening effort focused:

- Core banking ledger integration (no mainframe/COBOL bridge).
- Card network connectivity (Visa / Mastercard / payment processors).
- Physical branch network and on-premise hardware.
- Native mobile application code (the "mobile" surface is treated as a browser client only).

## 2. The 3-Tier Architecture

To ensure operational resilience and security isolation, the platform is strictly decoupled into three traditional tiers:

- **Presentation Layer (Web):** The client-facing interface, delivered as a browser-based SPA. This layer contains no business logic and no sensitive data at rest. It renders the UI, captures user input, and communicates exclusively with the Application Layer over HTTPS. The Presentation Layer can never reach the Data Layer directly.
- **Application Layer (API):** The central nervous system of the bank. This tier handles authentication, input validation, transaction processing, and orchestration. It acts as the strict intermediary between the Presentation and Data layers — no request from the client ever bypasses it.
- **Data Layer (Database):** The secure vault. This tier is dedicated entirely to data persistence, relational integrity, and encryption at rest (PostgreSQL native encryption + `pgcrypto` for column-level secrets). The Data Layer is isolated in a private subnet with no route to or from the public internet.

## 3. Trust Boundaries & Network Segmentation

In a secure architecture, we assume the network is hostile. We manage risk by defining strict trust boundaries — points where data moves from an untrusted zone to a trusted one.

- **The Untrusted Zone (Public Internet):** The starting point for all client traffic. We assume all data originating here is potentially malicious.
- **The Edge Boundary (DMZ):** The perimeter defense. Traffic from the internet must pass through this boundary before reaching our servers. The DMZ hosts the **WAF and reverse proxy** plus a **load balancer**; no application logic or data is resident here. It accepts only ports 80 (HTTP, redirect to HTTPS only) and 443 (HTTPS).
- **The Internal Boundary (App → Database):** The most critical trust boundary. The Application Layer operates in a restricted subnet and is the *only* entity permitted to open connections to the Data Layer. The Data Layer is **not internet-reachable** and has no public route — it is isolated in a private subnet, reachable only from the Application subnet.

## 4. Core Security Components

To defend these boundaries, SecureBank integrates three foundational security services:

- **Web Application Firewall (WAF):** Deployed at the Edge Boundary to inspect incoming HTTPS traffic. It drops malicious payloads (such as SQL injection, cross-site scripting, and brute-force attempts) before they ever reach the Application Layer.
- **Identity and Access Management (IAM):** Deployed as a **dedicated identity service within the Application subnet**, issuing short-lived JWT tokens consumed by the API. It enforces Multi-Factor Authentication (MFA), Role-Based Access Control (RBAC), and session lifecycle management (token expiry, refresh rotation, revocation).
- **Security Information and Event Management (SIEM):** The observability engine. It ingests telemetry from the WAF, the API, and the Data Layer, and it is the **authoritative append-only sink for audit logs** — application and database events are forwarded off-box so a compromise of the database cannot rewrite its own audit trail. The SIEM correlates events to detect anomalies and trigger automated SOC alerts.

## 5. Technology Stack

Our infrastructure relies on battle-tested, open-source technologies capable of enterprise-grade security hardening:

- **Frontend:** React 18 (browser-based SPA).
- **Backend API:** Python 3.12 with FastAPI.
- **Database:** PostgreSQL 15 — leveraging native RBAC, row-level security, `pgcrypto` for column-level encryption, and TLS for in-transit connections.
- **Operating System:** **Fedora 44** — the lab host and node OS. SELinux is enforced by default, providing mandatory access control at the kernel level. In a production analogue, this role would be filled by Fedora Server 44 or RHEL 9; the hardening procedures applied in Phase 4 are equivalent across the family.
- **Supporting services:** Secrets management via **HashiCorp Vault**; centralized logging via the SIEM (§4); encrypted off-site backups tested quarterly.