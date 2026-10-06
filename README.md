# -NETWORKWALKS-DANIEL-AGYEI-OMANTAM-B083-WK4-PM-CYBERSECURITY-LAB-SETUP
# Mediroza General Hospital — Penetration Testing Report

## Prepared for:  Mediroza General Hospital
## Prepared by:** [Daniel Agyei Omantam]
**Batch:** B083 | Week 4
**Target:** https://medirozahospital.com
**Test Type:** Black-box Penetration Test
**Duration:** 5 Days
**Authorization:** Written permission granted by client for all testing described below.

> This engagement was conducted in a controlled training environment for educational purposes. These techniques must never be applied to any system without explicit written permission from the owner.

---

## 1. Executive Summary

A black-box penetration test was conducted against Mediroza General Hospital's patient-facing web infrastructure. Testing uncovered a **critical authentication bypass** on the patient portal login page, which allowed unauthenticated access to confidential patient pathology reports. Further analysis of these reports revealed weak file-level encryption and sensitive metadata, which in turn led to the discovery of a **publicly accessible database backup** exposing the personal and financial data of all hospital staff, as well as shareholder ownership records.

Taken together, these findings represent a severe risk to patient confidentiality, staff privacy, and the hospital's regulatory and reputational standing. Several of the issues identified stem from basic, well-understood security practices that were not followed — unsanitized user input, weak password policies, and improper handling of legacy files on the production web server.

** Overall Risk to Client: Critical

---

## 2. Scope and Methodology

**In scope:** `medirozahospital.com` and its subpaths only.

**Rules of engagement:**
- No social engineering
- No denial-of-service testing
- No testing outside the agreed domain

**Methodology:** Testing followed a standard black-box approach — no credentials or internal documentation were provided in advance. The assessment proceeded through reconnaissance, vulnerability identification, exploitation, and post-exploitation analysis, consistent with a typical external attacker's process.

**Tools used:**
| Tool | Purpose |
|---|---|
| Browser Developer Tools | Manual request/response inspection |
| `curl` | Fetching pages and files directly from the command line |
| `pdfcrack` | Password recovery against encrypted PDF files |
| `exiftool` | Extracting hidden metadata from files |
| `qpdf` | Decrypting PDF files once a password was recovered |
| `grep` | Searching large text/database files for relevant data |

---

## 3. Findings and Proof of Exploitation

### Finding 1: SQL Injection — Authentication Bypass
**Location:** `/patient/login.php`
**Risk: Critical**

The patient portal login form passes user input directly into a database query without sanitization. Submitting a single quote (`'`) in the username field returned a raw database error, confirming the flaw:

```
Warning: mysqli_query(): You have an error in your SQL syntax; check the manual
that corresponds to your MySQL server version for the right syntax to use near
''' at line 1
```

This was then exploited using the payload:

```
Username: admin'-- 
Password: (any value)
```

The `--` sequence comments out the remainder of the query, including the password check, allowing login as the `admin` user with no valid password. This granted access to the patient portal, from which 3 confidential pathology reports were downloaded without authorization.


<img width="306" height="379" alt="Screenshot 2026-10-02 100758" src="https://github.com/user-attachments/assets/ff7bde07-3aa5-42a3-8d35-ae93a3d812c9" />

---

### Finding 2: Weak PDF Encryption Passwords
**Location:** Patient report PDFs retrieved from the portal
**Risk: High**

All 3 downloaded PDF reports were password-protected using standard PDF encryption (RC4, 128-bit). Using `pdfcrack` against a common password wordlist (rockyou.txt), all 3 passwords were recovered:

| File | Password |
|---|---|
| Report 1 | `1234` |
| Report 2 | `password` |
| Report 3 | `!@#$%^&` |

The first two were recovered almost instantly, confirming they are among the most common passwords in use globally. The third required a larger wordlist, indicating marginally better — but still inadequate — password strength.


<img width="1920" height="964" alt="image" src="https://github.com/user-attachments/assets/69c244a9-371b-4538-9eda-e1961d24f35e" />


---

### Finding 3: Sensitive Metadata Left in Patient PDFs
**Location:** `patient_report_3.pdf`
**Risk: Medium**

Metadata extracted from the decrypted PDF revealed internal information never intended for patient visibility:

```
Author   : j.malik
Comments : DB backup moved to /old before site migration, do not delete
```

This note, left by a staff member during file creation or editing, directly referenced an internal backup location on the production server.


<img width="1920" height="964" alt="image" src="https://github.com/user-attachments/assets/e7f0d499-f0ef-4892-963c-e49a4aa0b0a6" />


---

### Finding 4: Forgotten Backup Folder with Directory Listing Enabled
**Location:** `/old/`
**Risk: Critical**

The `/old/` path was listed in the site's `robots.txt` file during initial reconnaissance, and was later confirmed — via the metadata clue above — to contain a forgotten database backup. The folder has **directory listing enabled**, meaning its full contents are browsable by anyone who visits the URL directly:

```
https://medirozahospital.com/old/
→ mediroza_db_backup_2019.sql
```

This file was downloaded without any authentication required.

<img width="1920" height="964" alt="image" src="https://github.com/user-attachments/assets/c2d059ab-6bae-4b5a-984c-797b0f09c6c7" />

---

### Finding 5: Confidential Staff and Shareholder Data Exposed in Plain Text
**Location:** `old/mediroza_db_backup_2019.sql`
**Risk: Critical**

The exposed database backup contains, in unencrypted plain text:
- Full names, job titles, departments, email addresses, phone numbers, **national ID numbers**, and **monthly salaries** for all 18 hospital staff members
- Shareholder names, ownership percentages, shares held, and share class for 10 shareholders

**Sample (redacted):**

| Name | Job Title | Monthly Salary (ZAR) | National ID |
|---|---|---|---|
| Dr. Rajesh Naidoo | Chief Pathologist | 138,000 | 8502••••••••• |
| Sarah Botha | Chief Financial Officer | 152,000 | 8503••••••••• |
| Jameel Malik | IT Systems Administrator | 58,000 | 8510••••••••• |

| Shareholder | Share % | Shares Held | Class |
|---|---|---|---|
| Dr. Rajesh Naidoo | 18.0 | 180,000 | Ordinary |
| Cedar Health Holdings (Pty) Ltd | 15.0 | 150,000 | Ordinary |
| Dr. Johan van der Merwe | 12.0 | 120,000 | Ordinary |

*(Full dataset retained separately as evidence; not reproduced here in full to limit unnecessary exposure of personal data.)*

**Notable trail:** The `j.malik` author field found in Finding 3 corresponds to Jameel Malik, the IT Systems Administrator listed in the staff table — confirming the internal note was left by the person responsible for moving the backup.

---

## 4. Risk Rating Summary

| # | Vulnerability | Location | Risk |
|---|---|---|---|
| 1 | SQL injection login bypass | `/patient/login.php` | **Critical** |
| 2 | Weak PDF encryption passwords | `patient_report_*.pdf` | High |
| 3 | Sensitive metadata in patient PDFs | `patient_report_3.pdf` | Medium |
| 4 | Directory listing on forgotten backup folder | `/old/` | **Critical** |
| 5 | Staff salaries and shareholder data exposed in plain text | `old/mediroza_db_backup_2019.sql` | **Critical** |

---

## 5. Recommendations and Remediation

**SQL Injection (Finding 1)**
Replace raw SQL string concatenation with parameterized queries / prepared statements across all authentication and data-access code. Never build SQL queries using unsanitized user input.

**PDF Encryption (Finding 2)**
Enforce strong, unique passwords for any password-protected file distributed to patients, ideally system-generated rather than user-chosen. Consider replacing file-level PDF passwords with proper access-controlled delivery (e.g., authenticated download links) rather than relying on PDF encryption alone.

**Metadata Exposure (Finding 3)**
Strip all metadata from patient-facing documents before distribution, e.g. using `exiftool -all= filename.pdf` as part of the document generation pipeline.

**Directory Listing & Backup Exposure (Findings 4 & 5)**
- Disable directory listing on all web-accessible folders.
- Remove the `/old/` folder from the production web server entirely.
- Store database backups outside the web root, with access restricted to authorized internal systems only.
- Conduct a review of the server for any other legacy/forgotten files from past migrations.

---

## 6. Conclusion

This engagement identified a clear chain of failures — an exploitable login form, weak file passwords, careless metadata handling, and a forgotten backup left publicly accessible — that together resulted in the exposure of confidential patient reports, staff personal and financial data, and shareholder records. Addressing Finding 1 and Finding 4 should be treated as immediate priorities, given their critical severity and ease of exploitation.

---
*This document is produced for educational purposes as part of a Networkwalks training engagement. The target was authorised for security testing. These techniques must never be applied to any system without explicit written permission from the owner.*
