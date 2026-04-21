# Copilot Prompt: Generate LLD for Air Canada SFTP Transaction File Feature

---

## Prompt

You are a senior Java backend architect working inside the Finance Adaptor service.

Refer to the existing codebase to understand:
- Current Spring Batch job and step structure (Reader / Processor / Writer pattern)
- Partner config table and how bonus/credit-debit codes are stored per partner
- Quartz scheduling setup and how jobs are triggered
- OFB feeder file generation — existing pattern and flow

Using the above codebase context along with the following files in the /docs folder:
- docs/jira-story.md
- docs/requirements.md
- docs/file-spec-summary.md
- docs/data-model.md
- docs/Air Canada Billing File Requirements.xlsx

Generate a detailed Low Level Design (LLD) document for the Air Canada SFTP
Transaction File Generation feature.

---

## LLD Must Cover These Sections

1. **Overview & Scope**
   - What this feature does, what it does not do
   - How it fits into the existing Finance Adaptor service

2. **Component Design**
   - New Spring Batch Job → Steps → Reader / Processor / Writer
   - Follow the existing batch job pattern already in this repo
   - Show how Quartz triggers the batch job

3. **Key Class Design**
   - Interface names and their responsibilities
   - Key implementation classes
   - How config is injected and used per partner

4. **Sequence Diagram**
   - PlantUML format
   - Cover: Quartz trigger → Job → Read transactions → Process/map → Write CSV → Upload to S3

5. **Config Design**
   - What is config-driven: partner ID, bonus/credit-debit codes, S3 path, file name pattern
   - Config table structure (refer to data-model.md)
   - How a new partner can be onboarded with zero code change

6. **File Generation Logic**
   - Full column mapping from internal fields to Air Canada CSV headers
   - Field-level rules: InvoiceDate, InvoiceNumber, ActionType, CurrencyCode hardcoding
   - How multiple bonus codes per partner are handled
   - How BonVoyObtained and AmountDue are stripped of currency symbols

7. **S3 File Placement**
   - How the platform SFTP library is used to place the file into S3
   - Config-driven S3 path resolution

8. **Error Handling & Retry**
   - What happens if the batch job fails mid-way
   - How errors are surfaced (no silent failures)
   - Retry strategy if applicable

9. **Logging & Audit**
   - Log: job start, job end, record count processed, S3 path written, any errors
   - Follow existing logging pattern in this repo

10. **Assumptions & Out of Scope**
    - List all assumptions made
    - Explicitly call out what is out of scope

---

## Constraints

- No new database tables — use existing tables only
- No deployment or infrastructure details
- Quartz scheduling — follow the exact same pattern already used in this service
- OFB feeder file must not be touched or modified
- Keep code snippets minimal — only include for key logic such as:
  - Column mapping in the Processor
  - Batch Step configuration
  - InvoiceDate / InvoiceNumber calculation
- Output format: Markdown
- Be specific and precise — avoid generic Spring Batch boilerplate explanations
