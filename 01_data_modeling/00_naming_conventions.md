# SecureBank Lab - Database Naming Conventions

- **Version:** 1.0
- **Author:** Moad Morchid
- **Date:** 2026-10-16
- **Scope:** All data modeling artifacts (CDM, LDM, PDM), SQL schemas, database objects, and technical documentation across the SecureBank Lab project.

---

## 1. Purpose and Scope

This standard defines the naming rules for all entities, tables, columns, relationships, constraints, and indexes in the SecureBank Lab architecture. It applies across the full lifecycle: from conceptual modeling in SAP PowerDesigner (Phase 1) to physical implementation in PostgreSQL 15 (Phase 3).

Beyond basic readability, this convention serves as a schema-level security control: any reviewer or automated script must be able to identify PII and encrypted fields from the identifier name alone.

---

## 2. Naming Rules

| # | Object / Domain | Convention | Bad -> Good | Engineering Rationale |
|---|---|---|---|---|
| 1 | Conceptual Entity (CDM/LDM) | `PascalCase`, singular | `Client_Profiles` -> `ClientProfile` | Represents a domain concept in PowerDesigner diagrams before physical generation. |
| 2 | Physical Table (PDM/SQL) | `snake_case`, singular | `ClientProfile` / `clients` -> `client_profile` | Aligns with PostgreSQL lowercase identifier folding; eliminates double-quoting in SQL queries. |
| 3 | Column / Attribute | `snake_case`, lowercase | `ClientID` -> `client_id` | Standardizes physical attributes with table names and prevents case-sensitivity bugs. |
| 4 | Primary Key | `<singular_table>_id` | `id` -> `client_id` | Self-documenting in `JOIN` clauses (`USING (client_id)`) and unambiguous in query results. |
| 5 | Foreign Key Column | `<referenced_table>_id` | `fk_c` -> `account_id` | Identifies the target parent table immediately without inspecting DDL constraints. |
| 6 | Sensitive / Encrypted Column | Prefix `enc_` | `cin` -> `enc_cin` | Flags columns encrypted at rest (`pgcrypto`). Allows instant regex auditing of the PII surface. |
| 7 | Timestamps | Suffix `_at` (stored as `TIMESTAMPTZ` in UTC) | `created` -> `created_at` | Distinguishes temporal points from booleans/dates; UTC prevents timezone drift in audit logs. |
| 8 | Booleans | Prefix `is_`, `has_`, or `can_` | `active` -> `is_active`, `mfa` -> `has_mfa` | Reads naturally in `WHERE` clauses and prevents confusion with status enums. |
| 9 | Audit Columns | `created_by`, `updated_by`, `deleted_at` | `creator` -> `created_by` | Uniform lifecycle tracking across all auditable tables; supports soft-delete patterns. |
| 10 | Associative (Join) Table | Alphabetical `<table_a>_<table_b>` | `client_account` -> `account_client` | Alphabetical ordering (`a` before `c`) removes guesswork when querying many-to-many relationships. |
| 11 | Constraints and Indexes | `pk_`, `fk_`, `uq_`, `ck_`, `idx_` + `<table>_<col>` | `client_idx1` -> `idx_client_enc_cin` | Makes constraint violations in PostgreSQL error logs immediately traceable to the exact table and column. |
| 12 | Enum Values | `SCREAMING_SNAKE_CASE` | `'Active'` -> `'ACTIVE'` | Visually separates fixed state literals from SQL keywords and column identifiers. |
| 13 | Abbreviations | Whitelist only: `id`, `iban`, `cin`, `kyc`, `mfa`, `dpo`, `rssi` | `clt_iban` -> `client_iban` | Prevents cryptic shorthand while permitting standard banking and regional regulatory acronyms. |

**Note on the Transformation Pipeline (Rules 1-3):** The conceptual model uses `PascalCase` (Rule 1); SAP PowerDesigner automatically converts entities to `snake_case` during PDM generation (Rule 2). At no point should `PascalCase` and `snake_case` coexist in the same artifact.

**Order of Precedence:** When rules overlap, security prefixes (`enc_`) take priority over type prefixes. For example, an encrypted column always starts with `enc_` (e.g., `enc_iban`, `enc_mfa_secret`).

---

## 3. Schema-Level Security (enc_ Contract)

In SecureBank Lab, the `enc_` prefix is a technical contract, not a cosmetic label. Any attribute classified as Confidential or Secret (`cin`, IBANs, card tokens, KYC references, authentication secrets) must carry the `enc_` prefix and be encrypted at rest using PostgreSQL `pgcrypto` (AES-256).

This convention enforces three governance controls directly at the schema level:

1. **Automated PII Inventory (ISO/IEC 27001:2022 - A.5.34):** Auditors and CI scripts can enumerate every sensitive column in the database by querying `information_schema.columns` for `enc_%`. No external spreadsheet is required; the schema serves as its own live register.

2. **Data Leakage Prevention (ISO/IEC 27001:2022 - A.8.12):** In `JOIN` queries, API serializers, and log exports, `enc_cin` is immediately recognizable as restricted data. Developers and code reviewers can catch accidental exposure in pull requests without consulting a separate data dictionary.

3. **Privacy by Design (GDPR Art. 25 / Law 09-08):** Column-level encryption is defined during conceptual modeling (Phase 1) rather than retrofitted after security review. Encoding this decision in the column name ensures the requirement survives refactors and model regenerations.

Removing encryption or omitting the `enc_` prefix on a sensitive attribute requires a formal review before the model can be validated.

---

## 4. Enforcement and Exceptions

Compliance is verified across three stages of the engineering workflow:

- **Conceptual and Logical Modeling (Phase 1-2):** Checked during SAP PowerDesigner model reviews before PDM generation.
- **Physical Schema and CI (Phase 3):** Validated mechanically against `schema.sql`. A verification script (`naming_check.sql`) queries `information_schema` and `pg_catalog` using regex assertions to fail builds containing non-compliant identifiers.
- **Documentation:** All architecture diagrams, data dictionaries, and incident reports must match physical column and table names verbatim.

**Exceptions:** Any deviation from this standard must be documented as an Architecture Decision Record (ADR) under `docs/adr/`, detailing the rationale and alternatives considered. Silent deviations are treated as schema defects.

---

## 5. Revision History

| Version | Date | Author | Description |
|---|---|---|---|
| 1.0 | 2026-10-16 | Moad Morchid | Initial standard established for Phase 1-3 artifacts. |