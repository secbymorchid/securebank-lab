# Phase 1: Data Modeling & GRC Architecture

This directory contains the conceptual data model and governance documentation for **SecureBank Lab**. 

Before writing any SQL or provisioning PostgreSQL, the goal of this phase was to design the banking domain and its security controls together. In most projects, security and compliance are bolted on after the database is already built. Here, access control, auditability, and data protection are baked directly into the initial conceptual schema.

## Repository Contents

* **`SecureBank_CDM.cdm`** The SAP PowerDesigner Conceptual Data Model (15 entities, 14 relationships). I built this as a "rich" CDM: it preserves pure conceptual business rules and many-to-many relationships while already defining primary identifiers and conceptual data types to ensure a clean generation path toward the Logical and Physical models.
* **`diagrams/MCD.png`**  High-resolution export of the complete conceptual diagram (covering both the 9 core banking entities and the 6 security/SOC entities).
* **`00_naming_conventions.md`**  Our engineering standard for naming entities, attributes, and constraints. This isn't just for readability; it acts as a schema-level security control so automated scripts can immediately spot sensitive fields.
* **`classification.md`**  The GRC data classification policy. It defines our four sensitivity tiers (`Public`, `Internal`, `Confidential`, `Secret`) and maps them directly to **ISO/IEC 27001:2022** controls and **RGPD (GDPR)** requirements.
* **`data_dictionary.md`**  The complete attribute-by-attribute reference bridging the PowerDesigner diagram and the classification matrix. Every field across all 15 entities is documented with its data type, mandatory status, business owner, sensitivity tier, and specific ISO 27001 control.

## Architectural Decisions & Security-by-Design

Rather than treating the database as a passive storage layer, this model enforces three core security patterns from day one:

1. **Deterministic Encryption Governance (`enc_` Prefix):** Any attribute prefixed with `enc_` (such as `enc_cin`, `enc_iban`, `enc_card_number`, or `enc_password_hash`) is automatically governed as `Secret` under ISO 27001 A.8.24 and RGPD Art. 32. This makes sensitive columns impossible to miss during code reviews or physical database migration.
2. **Native RBAC & Session Tracking:** User authentication (`User`, `Session`) and authorization (`Role`, `Permission`) are modeled alongside retail banking entities (`Client`, `Account`, `Transaction`), ensuring strict separation of duties between customers, branch employees, and SOC staff.
3. **Tamper-Evident Auditing & SOC Triage:** The `AuditLog` entity is designed with cryptographic hash-chaining (`previous_hash` and `enc_log_hash`) so that any unauthorized database modification breaks the chain and triggers an investigation in the `Incident` table.
