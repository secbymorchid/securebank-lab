
# Phase 2: SecureBank Architecture & Trust Boundaries

This directory outlines the high-level system architecture, network segmentation, and security posture of the SecureBank infrastructure. 

Before diving into infrastructure-as-code or server configuration, it is critical to establish a shared mental model of how data flows through the system. This document defines the structural blueprint of SecureBank, illustrating how we separate concerns, enforce least privilege, and defend against external threats.

## 1. Architectural Context

SecureBank is a modern, digital-first retail banking platform designed to process sensitive financial transactions while withstanding targeted cyberattacks. The platform serves three distinct user groups, each with fundamentally different access requirements:
*   **Retail Clients:** Accessing the platform from public networks to view balances, initiate transfers, and manage cards.
*   **Branch Employees:** Accessing the platform from corporate networks to manage agency operations, perform KYC validations, and support clients.
*   **Security Operations Center (SOC) Analysts:** Accessing the system via dedicated secure terminals to monitor audit logs, investigate anomalies, and respond to active incidents.

## 2. The 3-Tier Architecture

To ensure operational resilience and security isolation, the platform is strictly decoupled into three traditional tiers:

*   **Presentation Layer (Web & Mobile):** The client-facing interface. This layer contains no business logic or sensitive data at rest. It simply renders the UI and captures user input, communicating exclusively over HTTPS.
*   **Application Layer (API):** The central nervous system of the bank. This tier handles authentication, input validation, transaction processing, and orchestration. It acts as the strict intermediary; the Presentation Layer can never bypass it to reach the data.
*   **Data Layer (Database):** The secure vault. This tier is dedicated entirely to data persistence, relational integrity, and cryptographic storage (as defined in our Phase 1 Conceptual Data Model).

## 3. Trust Boundaries & Network Segmentation

In a secure architecture, we assume the network is hostile. We manage risk by defining strict trust boundaries—points where data moves from an untrusted zone to a trusted one.

*   **The Untrusted Zone (Public Internet):** The starting point for all client traffic. We assume all data originating here is potentially malicious.
*   **The Edge Boundary (DMZ):** The perimeter defense. Traffic from the internet must pass through this boundary before reaching our servers. It is strictly limited to ports 80 (HTTP, redirect only) and 443 (HTTPS).
*   **The Internal Boundary (App to Database):** The most critical trust boundary. The Application Layer operates in a restricted subnet and is the *only* entity permitted to open connections to the Data Layer. The database is entirely air-gapped from the public internet.

## 4. Core Security Components

To defend these boundaries, SecureBank integrates three foundational security services:

*   **Web Application Firewall (WAF):** Deployed at the Edge Boundary to inspect incoming HTTPS traffic. It drops malicious payloads (such as SQL injection, Cross-Site Scripting, and brute-force attempts) before they ever touch the Application Layer.
*   **Identity and Access Management (IAM):** Centralized control for user authentication, Multi-Factor Authentication (MFA) enforcement, and Role-Based Access Control (RBAC). It governs session lifecycles and token issuance.
*   **Security Information and Event Management (SIEM):** The observability engine. It continuously ingests the cryptographic hash-chained audit logs from the database and telemetry from the WAF/API to detect anomalies and trigger automated SOC alerts.

## 5. Technology Stack

Our infrastructure relies on battle-tested, open-source technologies capable of enterprise-grade security hardening:

*   **Frontend:** React (Web) 
*   **Backend API:** Node.js or Python (FastAPI/Django)
*   **Database:** PostgreSQL 15 (leveraging advanced RBAC, row-level security, and cryptographic extensions)
*   **Operating System:** Ubuntu Server (LTS), serving as the hardened foundation for our application and database nodes.