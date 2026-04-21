# Requirements: Air Canada SFTP Transaction File Generation

## Story Reference
JIRA: [Your Story ID]  
Feature: Airline SFTP Transaction File Generation & Transfer — Air Canada

---

## 1. Objective

Generate a partner-specific transaction file for Air Canada containing all
funding transactions sent to OFB for a given billing month, and place the
generated file into a configured S3 location via the platform SFTP library.

---

## 2. Functional Requirements

### 2.1 File Generation
- Generate **one CSV file per billing month** scoped to Air Canada partner.
- File must contain all **funded transactions** sent to OFB for that billing month.
- File format: **CSV, comma-delimited**.
- Header row must use the **exact column names** defined in Column D of the
  Air Canada Billing File Requirements spec (Tab: Update Spec-Header Mapping).
- A partner can have **multiple bonus codes** (credit/debit codes). All
  transactions across all configured bonus codes for Air Canada must be included.

### 2.2 Field-Level Rules
- **InvoiceDate**: Defaults to the last day of the previous month. Format: `YYYY-MM-DD`.
- **InvoiceNumber**: Defaults to the last day of the previous month. Same value as InvoiceDate. Format: `YYYY-MM-DD`.
- **ActionType**: Always hardcoded to `BOOKING`.
- **CurrencyCode**: Always hardcoded to `USD`.
- **BonVoyObtained**: Plain numeric value — no `$` or currency symbol.
- **AmountDue**: Plain numeric value — no `$` or currency symbol.
- **DateTime**: Must be in Zulu (UTC) time format.
- **BookingID**: Integer value. Character length must be between 10 and 100.
- All other field mappings are defined in `file-spec-summary.md`.

### 2.3 Config-Driven Design
- Partner identity (Air Canada), associated bonus/credit-debit codes, S3 output
  path, and file naming pattern must all be **driven by partner config table**.
- No hardcoding of partner-specific values in code.
- New partners must be onboardable via config/DB change only — no code change.

### 2.4 S3 File Placement
- Generated file must be placed into the **S3 location** defined in partner config.
- File transfer is handled via the **existing platform SFTP library**.
- This service's responsibility ends at: generate CSV → place into S3.

### 2.5 OFB Feeder File
- The existing OFB feeder file generation must **continue without any changes**.
- This feature is additive — no modification to current OFB flow.

---

## 3. Scheduling
- Job is triggered via **Quartz scheduler**.
- Follow the same Quartz scheduling pattern already established in the
  Finance Adaptor service.
- Scheduling config (cron, trigger) is externalized — not hardcoded.

---

## 4. Non-Functional Requirements
- No new database tables are introduced.
- Solution must follow existing **Spring Batch job/step patterns** in Finance Adaptor.
- Logging must capture: job start/end, record count, S3 path written, any errors.
- Failed file generation must not silently succeed — errors must be surfaced.

---

## 5. Out of Scope
- SFTP server configuration or credentials management.
- Any changes to OFB feeder file logic.
- New database schema or migrations.
- Deployment or infrastructure setup.
- UI or API exposure of the generated file.

---

## 6. Assumptions
- Partner config table already holds Air Canada's partner ID and bonus codes.
- Platform SFTP library is available as a dependency in Finance Adaptor.
- Billing month is derived from the Quartz job trigger date (previous month).
- One Quartz job execution = one billing month's file.
