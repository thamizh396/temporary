# File Spec Summary: Air Canada SFTP Transaction File

## Source
Air Canada Billing File Requirements.xlsx — Tab: Update Spec-Header Mapping

---

## 1. File Format
- Format: CSV, comma-delimited
- Encoding: UTF-8
- One file per billing month
- SFTP hosted by Air Canada
- File placed into S3 via platform SFTP library

---

## 2. CSV Header Row (Column Order)

```
AuthorizationID,BookingID,DateTime,ActionType,BonVoyObtained,CurrencyCode,AmountDue,InvoiceDate,InvoiceNumber,RPP
```

---

## 3. Column Mapping

| # | CSV Header      | Air Canada Field Name       | Internal Field Mapping         | Rules / Notes                                             |
|---|-----------------|-----------------------------|--------------------------------|-----------------------------------------------------------|
| 1 | AuthorizationID | AuthorizationID             | TRANS_ID                       | Air Canada's Transaction ID                               |
| 2 | BookingID       | BookingID                   | ACCOUNT_BON_ID                 | Marriott Transaction ID. Integer. Min: 10, Max: 100 chars |
| 3 | DateTime        | BookingDateTime             | Date & Time of Points Issuance | Zulu (UTC) time format                                    |
| 4 | ActionType      | TransactionType             | Hardcoded: `BOOKING`           | Always static — no DB mapping needed                      |
| 5 | BonVoyObtained  | BonVoyObtained              | # Points Issued                | Numerical value only — no `$` sign                        |
| 6 | CurrencyCode    | CurrencyCode                | Hardcoded: `USD`               | Always static — no DB mapping needed                      |
| 7 | AmountDue       | AmountDue                   | Calculated Billing Amount      | Numerical value only — no `$` sign                        |
| 8 | InvoiceDate     | InvoiceDate                 | Invoice Date                   | Last day of previous billing month. Format: `YYYY-MM-DD`  |
| 9 | InvoiceNumber   | InvoiceNumber               | Invoice Date                   | Same value as InvoiceDate. Format: `YYYY-MM-DD`           |
|10 | RPP             | Rate Per Point              | Rate                           | Numerical value                                           |

---

## 4. Excluded Fields (Confirmed Not Required by Air Canada)

| Air Canada Header              | Reason                               |
|--------------------------------|--------------------------------------|
| MemberNumber (Aeroplan #)      | Confirmed not required by Air Canada |
| CancellationDateTimeDescoped   | Confirmed not required by Air Canada |
| PointsRedeemedDescoped         | Confirmed not required by Air Canada |

---

## 5. Field-Level Rules

| Field           | Rule                                                                          |
|-----------------|-------------------------------------------------------------------------------|
| InvoiceDate     | Last day of the **previous month**. Format: `YYYY-MM-DD`                      |
| InvoiceNumber   | Same value as InvoiceDate. Format: `YYYY-MM-DD`                               |
| ActionType      | Always hardcoded to `BOOKING`                                                 |
| CurrencyCode    | Always hardcoded to `USD`                                                     |
| BonVoyObtained  | Plain numeric value — strip any `$` or currency symbol                        |
| AmountDue       | Plain numeric value — strip any `$` or currency symbol                        |
| DateTime        | Must be in Zulu (UTC) time format                                             |
| BookingID       | Integer value. Character length must be between 10 and 100                    |
| AuthorizationID | String — Air Canada's own transaction reference ID                            |
| RPP             | Numeric — Rate per point value                                                |

---

## 6. Sample Row (from Spec)

```
AuthorizationID,BookingID,DateTime,ActionType,BonVoyObtained,CurrencyCode,AmountDue,InvoiceDate,InvoiceNumber,RPP
44000020465274,1701387250,2025-10-01T00:17:057,BOOKING,13000,USD,123.5,2025-11-02,2025-11-02,0.0095
```

---

## 7. High Level Requirements (from Spec)
- Maintain existing file structure and format
- SFTP is hosted by Air Canada
- Air Canada can provide UAT SFTP details for testing
- File must be `.csv` format, comma-delimited
- Marriott to complete a form to provide whitelisted IPs and decryption keys
