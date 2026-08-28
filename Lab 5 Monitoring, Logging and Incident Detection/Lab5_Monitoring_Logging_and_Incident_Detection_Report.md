# Lab 5:Monitoring, Logging and Incident Detection

**Course:** Cloud Computing Security Essentials  
**Lab:** 5 Monitoring, Logging and Incident Detection  
**Date:** 3 September 2026

## 1. Introduction and objectives

This lab demonstrated how operational logs support cloud security monitoring, incident detection, digital forensics, and compliance evidence. A set of authentication and data-export records was generated locally, centralised in a LocalStack implementation of CloudWatch Logs, queried for suspicious activity, protected using a SHA-256 hash chain, and then correlated into an incident alert. The final activity applied a model containment rule and preserved a hashed evidence copy.

The work addressed the lab learning outcomes: centralising cloud telemetry, distinguishing logs from events, detecting tampering, correlating indicators into an incident, and applying the detect–contain–collect-evidence–document response lifecycle.

## 2. Lab environment and setup

LocalStack was started in Docker and exposed on port `4566`. The AWS CLI was configured to use the LocalStack endpoint, then the `/ccse/app` log group and its `auth` stream were created.

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack
EP='--endpoint-url=http://localhost:4566'
aws $EP logs create-log-group --log-group-name /ccse/app
aws $EP logs create-log-stream --log-group-name /ccse/app --log-stream-name auth
```

Evidence: [LocalStack startup](Evidence/0.%20Start%20LocalStack%20container.png) and [log-group/log-stream creation](Evidence/0.2.%20Create%20log%20group%20and%20log%20stream.png).

## 3. Session A — Logging and centralisation

### Task 1: Generate application logs

An `auth.log` file was created with seven timestamped application records. It contains one legitimate login by `ahmad`, four failed attempts against `admin` from `203.0.113.9`, a later successful login for `admin` from the same address, and a `500MB` data export.

| Time | Activity | User | Source IP | Security interpretation |
| --- | --- | --- | --- | --- |
| 09:00:01 | `LOGIN_OK` | ahmad | 10.0.0.5 | Normal internal login |
| 09:01:10–09:01:18 | `LOGIN_FAIL` × 4 | admin | 203.0.113.9 | Repeated credential guessing/brute-force indicator |
| 09:01:22 | `LOGIN_OK` | admin | 203.0.113.9 | Suspicious success after failures |
| 09:01:40 | `EXPORT_DATA`, `size=500MB` | admin | 203.0.113.9 | Potential large-scale data exfiltration |

Evidence: [generated `auth.log`](Evidence/1.%20Generate%20Application%20Logs.png).

### Task 2: Centralise logs in CloudWatch Logs

Each line in `auth.log` was sent to `/ccse/app` / `auth` using `aws logs put-log-events`. The central copy was then retrieved using `get-log-events`.

```bash
TS=$(date +%s000)
while IFS= read -r line; do
  aws $EP logs put-log-events --log-group-name /ccse/app --log-stream-name auth \
    --log-events timestamp=$TS,message="$line" >/dev/null
  TS=$((TS+1000))
done < auth.log

aws $EP logs get-log-events --log-group-name /ccse/app --log-stream-name auth \
  --query 'events[].message' --output text
```

The read-back confirms that the locally generated records were available in the central log service. Centralisation is important because it gives defenders a single searchable source and prevents analysis from depending only on logs left on one application host.

Evidence: [CloudWatch/LocalStack read-back](Evidence/2.%20Centralise%20Logs%20%28Ship%20to%20CloudWatch%29.png).

### Task 3: Query security-relevant activity

The following query groups failed login records by their fields:

```bash
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

The sample data contains **four failed login attempts** from `ip=203.0.113.9` against `user=admin`. This meets the suspicious-failure threshold later used by the correlation rule.

A **log** is a durable, timestamped record of an occurrence, for example `LOGIN_FAIL user=admin ip=203.0.113.9`. An **event** is a meaningful condition or trigger derived from one or more records, for example: “four failures from `203.0.113.9`; raise an alert.”

Evidence: [failed-login query](Evidence/3.%20Query%20for%20Security-Relevant%20Activity.png).

## 4. Session B — Tamper-proofing, detection and response

### Task 4: Create a tamper-evident hash chain

Each line was chained to the previous SHA-256 digest, starting with `PREV=0`. The digest of a line therefore depends on all preceding lines.

```bash
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s\n' "$line" "$PREV"
done < auth.log > auth.chain
```

The export size was then modified from `500MB` to `5MB` in `auth.tampered`. Recomputing the chain produced a different final chain value and the console reported `TAMPER DETECTED: Hash chain broken!`. Since each hash includes the previous hash, altering even one record changes that record’s digest and all subsequent digests. The final digest should also be stored or forwarded to a separate append-only location; otherwise an attacker able to alter the application log may be able to rewrite the local chain as well.

Evidence: [hash-chain creation](Evidence/4.%20Tamper-Proof%20%28Hash-Chained%29%20Logs.png) and [tamper-detection result](Evidence/4.1.%20Tamper-Proof%20%28Hash-Chained%29%20Logs.png).

> Note: the captured tamper-verification terminal output includes an `awk` regular-expression error while extracting the displayed original final hash. The key result is still visible—the recomputed tampered hash differs and the script reports that tampering was detected. For a clean re-run, extract the final hash with `awk -F'|' '{print $2}' auth.chain | tail -n 1` and compare it with the recomputed value.

### Task 5: Correlate records to detect an incident

The detection logic counted failures, successes, and exports for the suspicious address.

```bash
IP=203.0.113.9
FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)
SUCCESS=$(grep -c "LOGIN_OK.*$IP" auth.log)
EXPORT=$(grep -c "EXPORT_DATA.*$IP" auth.log)
echo "IP=$IP fails=$FAILS success=$SUCCESS export=$EXPORT"
```

Observed output:

```text
IP=203.0.113.9 fails=4 success=1 export=1
ALERT: probable brute-force -> compromise -> data exfiltration
```

No single record proves the incident by itself. Four failures suggest credential guessing; the subsequent successful `admin` login suggests the attack may have succeeded; and the immediate 500MB export provides a potential impact indicator. Correlating the shared IP address and sequence turns these separate observations into a probable brute-force compromise followed by data exfiltration—the type of multi-event detection normally performed by a SIEM.

Evidence: [correlation alert](Evidence/5.%20Detect%20the%20Incident%20%28Correlation%29.png).

### Task 6: Containment and evidence collection

Containment was modelled by applying an `iptables` rule inside a privileged Alpine container to drop traffic from `203.0.113.9`.

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c \
  'apk add -q iptables; iptables -A INPUT -s 203.0.113.9 -j DROP; iptables -L INPUT -n | tail -2'
```

The displayed rule is `DROP all -- 203.0.113.9 0.0.0.0/0`, showing the attacker address was blocked in the lab model. A timestamped evidence copy was then created and hashed:

```bash
cp auth.log evidence_$(date +%Y%m%d).log
sha256sum evidence_*.log > evidence.sha256
sha256sum -c evidence.sha256
```

The evidence file was `evidence_20260828.log`. Its recorded SHA-256 digest was:

```text
0adc5d2ac06cbbdd366099bcc0540c4c0f76946e71b52e4c99322731696a203b
```

The verification result `evidence_20260828.log: OK` confirms that the saved evidence was unchanged at the time it was verified.

Evidence: [containment and evidence hash](Evidence/6.%20Incident%20Response.png) and [verification output](Evidence/7.%20Verification%20Command.png).

## 5. Incident report

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

## 6. Incident timeline

| Timestamp | Observation | Response significance |
| --- | --- | --- |
| 09:01:10 | First failed `admin` login from `203.0.113.9` | Start of suspicious authentication activity |
| 09:01:12 | Second failed login | Repetition increases suspicion |
| 09:01:15 | Third failed login | Correlation threshold reached |
| 09:01:18 | Fourth failed login | Strong brute-force indicator |
| 09:01:22 | Successful `admin` login from same IP | Possible account compromise |
| 09:01:40 | 500MB export by same IP | Potential data exfiltration |
| After detection | IP blocked; log copied and hashed | Containment and preservation of evidence |

## 7. Answers to short-answer questions

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

## 8. Verification and best-practices checklist

The following commands confirmed the central log group and evidence-file integrity:

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
sha256sum -c evidence.sha256
```

- [x] Logs were centralised in `/ccse/app` rather than left only on the application host.
- [x] Failed logins were identified and grouped by source IP.
- [x] A SHA-256 hash chain was used to make log changes evident.
- [x] Multiple records were correlated into a security alert.
- [x] The attacker IP was contained in the lab model.
- [x] A timestamped evidence copy and checksum were created and verified.
- [x] The incident findings, timeline, and lesson learned were documented.

## 9. Conclusion

The lab achieved an end-to-end monitoring and incident-response workflow: generate logs, centralise them, identify suspicious authentication activity, protect their integrity, correlate a probable compromise, contain the source, and preserve verifiable evidence. This workflow supports the course objective of secure cloud operations that safeguard data integrity.
