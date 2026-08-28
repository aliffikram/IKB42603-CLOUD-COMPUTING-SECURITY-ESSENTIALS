# Lab 5:Monitoring, Logging and Incident Detection

**Course:** Cloud Computing Security Essentials  
**Lab:** 5 Monitoring, Logging and Incident Detection  
**Date:** 3 September 2026

## Objectives

This lab demonstrated how operational logs support cloud security monitoring, incident detection, digital forensics, and compliance evidence. A set of authentication and data-export records was generated locally, centralised in a LocalStack implementation of CloudWatch Logs, queried for suspicious activity, protected using a SHA-256 hash chain, and then correlated into an incident alert. The final activity applied a model containment rule and preserved a hashed evidence copy.

The work addressed the lab learning outcomes: centralising cloud telemetry, distinguishing logs from events, detecting tampering, correlating indicators into an incident, and applying the detect–contain–collect-evidence–document response lifecycle.

## Lab environment and setup

LocalStack was started in Docker and exposed on port `4566`. The AWS CLI was configured to use the LocalStack endpoint, then the `/ccse/app` log group and its `auth` stream were created.


<img width="439" height="23" alt="0  Start LocalStack container" src="https://github.com/user-attachments/assets/9da355fd-2342-40eb-b573-0388b964b54e" />       

<img width="310" height="13" alt="0 1  Set endpoint variable" src="https://github.com/user-attachments/assets/47591c3d-ad5c-4f94-b45d-ca22ab8533a0" />

<img width="405" height="22" alt="0 2  Create log group and log stream" src="https://github.com/user-attachments/assets/214a90f2-f961-403f-b472-f6bd3607bb2f" />


## Session A — Logging and centralisation

### Task 1: Generate application logs

An `auth.log` file was created with seven timestamped application records. It contains one legitimate login by `ahmad`, four failed attempts against `admin` from `203.0.113.9`, a later successful login for `admin` from the same address, and a `500MB` data export.


<img width="350" height="188" alt="1  Generate Application Logs" src="https://github.com/user-attachments/assets/562351b7-6af5-4c58-89c0-d37be6419888" />


| Time | Activity | User | Source IP | Security interpretation |
| --- | --- | --- | --- | --- |
| 09:00:01 | `LOGIN_OK` | ahmad | 10.0.0.5 | Normal internal login |
| 09:01:10–09:01:18 | `LOGIN_FAIL` × 4 | admin | 203.0.113.9 | Repeated credential guessing/brute-force indicator |
| 09:01:22 | `LOGIN_OK` | admin | 203.0.113.9 | Suspicious success after failures |
| 09:01:40 | `EXPORT_DATA`, `size=500MB` | admin | 203.0.113.9 | Potential large-scale data exfiltration |



### Task 2: Centralise logs in CloudWatch Logs

Each line in `auth.log` was sent to `/ccse/app` / `auth` using `aws logs put-log-events`. The central copy was then retrieved using `get-log-events`.

<img width="601" height="220" alt="2  Centralise Logs (Ship to CloudWatch)" src="https://github.com/user-attachments/assets/2cdda754-2ce8-47f7-8d48-921ae3818c43" />

The read-back confirms that the locally generated records were available in the central log service. Centralisation is important because it gives defenders a single searchable source and prevents analysis from depending only on logs left on one application host.

### Task 3: Query security-relevant activity

The following query groups failed login records by their fields:

<img width="409" height="22" alt="3  Query for Security-Relevant Activity" src="https://github.com/user-attachments/assets/0a2dbb0a-dd28-42a0-9527-38b5dfea0e0b" />


The sample data contains **four failed login attempts** from `ip=203.0.113.9` against `user=admin`. This meets the suspicious-failure threshold later used by the correlation rule.

A **log** is a durable, timestamped record of an occurrence, for example `LOGIN_FAIL user=admin ip=203.0.113.9`. An **event** is a meaningful condition or trigger derived from one or more records, for example: “four failures from `203.0.113.9`; raise an alert.”

## 4. Session B — Tamper-proofing, detection and response

### Task 4: Tamper-Proof (Hash-Chained) Logs

Each line was chained to the previous SHA-256 digest, starting with `PREV=0`. The digest of a line therefore depends on all preceding lines.

<img width="601" height="352" alt="4  Tamper-Proof (Hash-Chained) Logs" src="https://github.com/user-attachments/assets/44fc48bb-6fd9-42fc-aae9-51777dbb7de0" />

<img width="600" height="75" alt="4 1  Tamper-Proof (Hash-Chained) Logs" src="https://github.com/user-attachments/assets/233e9875-abae-4d20-84f2-7ff034497b78" />


The export size was then modified from `500MB` to `5MB` in `auth.tampered`. Recomputing the chain produced a different final chain value and the console reported `TAMPER DETECTED: Hash chain broken!`. Since each hash includes the previous hash, altering even one record changes that record’s digest and all subsequent digests. The final digest should also be stored or forwarded to a separate append-only location; otherwise an attacker able to alter the application log may be able to rewrite the local chain as well.



### Task 5: Detect the Incident (Correlation)

The detection logic counted failures, successes, and exports for the suspicious address.

<img width="373" height="133" alt="5  Detect the Incident (Correlation)" src="https://github.com/user-attachments/assets/134ae989-90c2-4cab-a3f6-81de73034378" />


No single record proves the incident by itself. Four failures suggest credential guessing; the subsequent successful `admin` login suggests the attack may have succeeded; and the immediate 500MB export provides a potential impact indicator. Correlating the shared IP address and sequence turns these separate observations into a probable brute-force compromise followed by data exfiltration—the type of multi-event detection normally performed by a SIEM.


### Task 6: Incident Response


<img width="487" height="89" alt="6  Incident Response" src="https://github.com/user-attachments/assets/f7cdfc0d-5aba-4105-9125-6d46c0616357" />


Containment was modelled by applying an `iptables` rule inside a privileged Alpine container to drop traffic from `203.0.113.9`.


The displayed rule is `DROP all -- 203.0.113.9 0.0.0.0/0`, showing the attacker address was blocked in the lab model. A timestamped evidence copy was then created and hashed:


The verification result  confirms that the saved evidence was unchanged at the time it was verified.

## Incident report

### Detection

The monitoring rule detected a probable security incident involving `203.0.113.9`. Correlation produced `fails=4`, `success=1`, and `export=1`, then raised the alert `probable brute-force -> compromise -> data exfiltration`.

### Analysis

The timeline begins with four consecutive failed attempts to access the `admin` account between 09:01:10 and 09:01:18. At 09:01:22, the same external IP successfully logged in as `admin`. Eighteen seconds later it requested a 500MB data export. The order, common IP, privileged account, and large export make accidental activity unlikely and indicate a probable brute-force compromise followed by attempted exfiltration.

### Containment

Traffic from `203.0.113.9` was blocked using an `iptables` `DROP` rule in the lab containment environment. In a production response, the account credentials would also be reset, active sessions revoked, and the block applied at the relevant firewall, security group, or web-application layer.

### Evidence and integrity

The original log was copied to the timestamped evidence file `evidence_20260828.log` and protected with a SHA-256 checksum recorded in `evidence.sha256`. The checksum verification returned `OK`. A separate SHA-256 hash chain was also generated for the original log; changing the export amount broke the expected chain result and raised a tamper-detection message. Together, these controls help preserve the integrity and evidential value of the records.

### Lesson learned

Security monitoring needs both central visibility and correlation. A single failed login may be benign, but a burst of failures, a success, and a high-volume export from one IP form a clear incident pattern. Centralised, tamper-evident logs allow that pattern to be detected and provide reliable evidence for investigation and compliance.

## Incident timeline

| Timestamp | Observation | Response significance |
| --- | --- | --- |
| 09:01:10 | First failed `admin` login from `203.0.113.9` | Start of suspicious authentication activity |
| 09:01:12 | Second failed login | Repetition increases suspicion |
| 09:01:15 | Third failed login | Correlation threshold reached |
| 09:01:18 | Fourth failed login | Strong brute-force indicator |
| 09:01:22 | Successful `admin` login from same IP | Possible account compromise |
| 09:01:40 | 500MB export by same IP | Potential data exfiltration |
| After detection | IP blocked; log copied and hashed | Containment and preservation of evidence |

## Answers to short-answer questions

### Q1. What is the difference between a log and an event? Give an example of each.

A log is a durable record of an individual occurrence. In this lab, `2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9` is a log entry. An event is a condition or notification generated from activity, often in near real time. The correlation alert for four failed logins followed by a success and export is an event.

### Q2. Why must audit logs be tamper-proof, and how does a hash chain achieve this?

Audit logs must be tamper-proof so attackers cannot erase or modify evidence of their actions, and so investigators and auditors can trust the records. In a hash chain, each entry’s hash is calculated from its content plus the previous hash. Editing one entry changes its hash and invalidates every later link, making unauthorised alteration detectable. Keeping the final hash in a separate append-only store strengthens this control.

### Q3. How did correlation detect an incident that no single log line revealed?

Correlation linked four failed logins, a later successful `admin` login, and a 500MB export by the same IP address. Each line alone has alternative explanations; their sequence and common source indicate a likely brute-force attack, compromise, and exfiltration.

### Q4. List the incident-response steps performed and the goal of each.

| Step | Action performed | Goal |
| --- | --- | --- |
| Detect | Counted failures, success and export, then raised an alert | Identify the probable incident |
| Contain | Dropped traffic from `203.0.113.9` | Stop further activity from the suspected source |
| Collect | Copied `auth.log` to a dated evidence file | Preserve the original records for investigation |
| Verify integrity | Created and checked `evidence.sha256`; used a hash chain | Demonstrate evidence has not been altered |
| Document | Recorded findings and timeline in this report | Support investigation, recovery, and lessons learned |

### Q5. How do the same logs serve both security monitoring and compliance evidence?

For security monitoring, the logs provide near-real-time visibility and the data needed to detect suspicious patterns. For compliance, central retention, timestamps, integrity checks, and documented response actions provide auditable proof that security-relevant activity was recorded, investigated, and handled with controlled evidence.

## Verification and best-practices checklist

The following commands confirmed the central log group and evidence-file integrity:

<img width="423" height="173" alt="7  Verification Command" src="https://github.com/user-attachments/assets/fda96c25-af92-4908-b2d8-c387d26fc966" />

## Cleanup & Teardown

<img width="459" height="44" alt="8  Cleanup   Teardown" src="https://github.com/user-attachments/assets/24e540de-56c3-4c13-89b1-ac555d34f5c5" />


- [x] Logs were centralised in `/ccse/app` rather than left only on the application host.
- [x] Failed logins were identified and grouped by source IP.
- [x] A SHA-256 hash chain was used to make log changes evident.
- [x] Multiple records were correlated into a security alert.
- [x] The attacker IP was contained in the lab model.
- [x] A timestamped evidence copy and checksum were created and verified.
- [x] The incident findings, timeline, and lesson learned were documented.

## Conclusion

The lab achieved an end-to-end monitoring and incident-response workflow: generate logs, centralise them, identify suspicious authentication activity, protect their integrity, correlate a probable compromise, contain the source, and preserve verifiable evidence. This workflow supports the course objective of secure cloud operations that safeguard data integrity.
