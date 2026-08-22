# Lab 3: Data Protection, Encryption and Key Management

**Course:** Cloud Computing Security Essentials  
**Lab:** 3 Data Protection, Encryption and Key Management  
**Date:** 20 August 2026

## Objective

This lab demonstrates how encryption and key management protect cloud data. The objectives are to encrypt data at rest with AES, use RSA for public-key encryption and digital signatures, protect data in transit with TLS, use LocalStack KMS for envelope encryption, apply separate tenant keys and cryptographic erasure, and verify integrity through hashing and a tamper-evident hash chain.

## Lab Overview

The lab is divided into two sessions:

- **Session A:** symmetric encryption, asymmetric encryption and digital signatures, and TLS protection for data in transit.
- **Session B:** KMS keys, envelope encryption, per-tenant keys, cryptographic erasure, and integrity verification.

The environment uses OpenSSL, Docker, `curl`, the AWS CLI pointed at LocalStack KMS, and SHA-256 hashing utilities.

## Session A: Encryption Fundamentals

### Task 1: Symmetric Encryption for Data at Rest

A sensitive patient record was created and encrypted with AES-256. Viewing the encrypted file showed unreadable ciphertext. The same passphrase was then used to decrypt the file, and the `diff` result confirmed that the decrypted record matched the original.

![Create a sample sensitive record](Evidence/1.%20Create%20a%20sample%20sensitive%20record.png)

![Encrypt the record with AES-256](Evidence/1.1.%20Encrypt%20with%20AES-256.png)

![Encrypted ciphertext is unreadable](Evidence/1.2.%20Prove%20it%20is%20unreadable.png)

![Successful AES-256 decryption](Evidence/1.3.%20Decrypt%20back.png)

AES provides fast confidentiality for stored data. However, the shared symmetric key must be protected because anyone who obtains it can decrypt the record.

### Task 2: Asymmetric Encryption and Digital Signatures

A 2048-bit RSA public/private key pair was generated. The public key encrypted the record, while the matching private key decrypted it. A SHA-256 signature was then created with the private key and successfully verified with the public key.

![Generate the RSA key pair](Evidence/2.%20Generate%20a%202048-bit%20key%20pair.png)

![Encrypt with the public key and decrypt with the private key](Evidence/2.1.%20Encrypt%20with%20the%20PUBLIC%20key%2C%20decrypt%20with%20the%20PRIVATE%20key.png)

![Sign with the private key and verify with the public key](Evidence/2.2.%20Sign%20with%20the%20PRIVATE%20key%3B%20verify%20with%20the%20PUBLIC%20key.png)

The encryption test demonstrates confidentiality without sharing the private key. The successful signature verification demonstrates integrity and confirms that the signature was created by the private-key holder.

### Task 3: Encryption in Transit with TLS

A self-signed certificate was generated and used by a small NGINX container serving HTTPS on port 8443. A `curl -k` request retrieved the record through the TLS-protected channel. The `-k` option was necessary only because the local certificate was self-signed and not trusted by a public certificate authority.

![Generate a self-signed certificate](Evidence/3.%20Generate%20a%20self-signed%20certificate.png)

![Serve HTTPS from the container](Evidence/3.1.%20Serve%20HTTPS%20on%20port%208443%20using%20a%20small%20container.png)

![Connect to the service over TLS](Evidence/3.2.%20Connect%20over%20TLS%20%28-k%20accepts%20the%20self-signed%20cert%29.png)

TLS protects the record while it travels over the network. Without TLS, an on-path attacker could read a plaintext HTTP request. In production, the server should use a trusted certificate and clients must validate the certificate rather than using `-k`.

## Session B: Key Management, Envelope Encryption and Integrity

### Task 4: Create and Use a KMS Master Key

A customer-managed KMS key for tenant A was created in LocalStack, and its KeyId was assigned to `KEY_A`. The KMS key was used to encrypt a small secret directly.

![KMS master-key task](Evidence/4.%20Create%20and%20Use%20a%20KMS%20Master%20Key.png)

![Create tenant A's KMS key and capture its KeyId](Evidence/4.1.%20Create%20a%20customer%20master%20key%20%28CMK%29%20and%20capture%20its%20KeyId.png)

![Assign the KeyId to KEY_A](Evidence/4.2.%20Copy%20the%20KeyId%20from%20the%20output%20into%20KEY_A%20below.png)

![Encrypt a small secret directly with KMS](Evidence/4.3.%20Encrypt%20a%20small%20secret%20directly%20with%20KMS.png)

KMS centralises key lifecycle and access controls. Direct KMS encryption is suitable for small values; large application files should use envelope encryption.

### Task 5: Envelope Encryption

KMS generated an AES-256 data key in plaintext and encrypted forms. The plaintext data key was decoded and used locally to encrypt the record. It was then deleted from disk, leaving only the KMS-wrapped data key and the encrypted record.

![Generate a plaintext and wrapped data key](Evidence/5.%20Ask%20KMS%20for%20a%20data%20key%20%28returns%20plaintext%20%2B%20encrypted%20versions%29.png)

![Encrypt the record locally with the plaintext data key](Evidence/5.1.%20Encrypt%20the%20big%20file%20locally%20with%20the%20PLAINTEXT%20data%20key.png)

![Remove the plaintext data key from disk](Evidence/5.2.%20Destroy%20the%20plaintext%20data%20key%20from%20disk%20%E2%80%94%20keep%20only%20the%20wrapped%20copy.png)

Envelope encryption uses a fast local data-encryption key for the file while KMS protects the smaller data key. To decrypt later, an authorised client asks KMS to unwrap the stored encrypted data key, uses it briefly, and then discards it again.

### Task 6: Per-Tenant Keys and Cryptographic Erasure

A separate KMS key was created for tenant B and assigned to `KEY_B`, keeping it distinct from tenant A's key. Tenant A's key was scheduled for deletion and disabled immediately. After the key was disabled, an attempt to decrypt tenant A's wrapped data key failed.

![Create a separate KMS key for tenant B](Evidence/6.%20A%20separate%20key%20for%20tenant%20B.png)

![Assign the KeyId to KEY_B](Evidence/6.1.%20Copy%20the%20KeyId%20from%20the%20output%20into%20KEY_B%20below.png)

![Schedule deletion and disable tenant A's key](Evidence/6.2.%20Schedule%20deletion%20of%20tenant%20A%27s%20key%20%28min%20window%29%20%26%20Disable%20it%20immediately%20to%20simulate%20erasure.png)

![Decrypt fails after tenant A's key is disabled](Evidence/6.3.%20Attempt%20to%20unwrap%20tenant%20A%27s%20data%20key%20now%20%E2%80%94%20it%20should%20FAIL.png)

This proves cryptographic erasure: ciphertext and backups may remain, but without the KMS key they cannot be decrypted. Separate keys also reduce the impact of a key compromise to the affected tenant.

### Task 7: Integrity and Tamper-Evident Logging

A SHA-256 hash was calculated for the original record. After modifying a copy, its hash differed from the original hash. A simple hash chain was also generated, with every entry containing the previous hash.

![Calculate the SHA-256 fingerprint](Evidence/7.%20Fingerprint%20the%20file.png)

![Tampered copy produces a different hash](Evidence/7.1.%20Tamper%20with%20a%20copy%20and%20show%20the%20hash%20changes.png)

![Hash chain for a tamper-evident log](Evidence/7.2.%20Hash%20chain%20each%20entry%20includes%20the%20previous%20hash%20%28tamper-evident%20log%29.png)

Encryption protects confidentiality, whereas hashing protects integrity. The changed hash makes file tampering detectable, and the chain makes changes to earlier log entries visible in subsequent entries.

## Short-Answer Questions

### Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.

Symmetric encryption uses one shared secret for encryption and decryption. It is fast and is suitable for bulk data such as files, databases, and backups. Its challenge is secure key distribution because every authorised party must obtain and protect the same secret.

Asymmetric encryption uses a public/private key pair. It is slower, but the public key can be distributed openly while the private key remains secret. It is used for identity, certificates, signatures, and protecting or exchanging small symmetric data keys. Hybrid encryption combines both approaches: asymmetric cryptography protects the data key and symmetric cryptography encrypts the large data.

### Q2. Why is key management described as the weakest link, not the algorithm?

Strong algorithms such as AES-256 and RSA provide little protection when their keys are exposed. Attackers often target keys in source code, logs, backups, files, or overly broad cloud permissions rather than breaking the algorithm. Key management covers key creation, secure storage, access policies, rotation, auditing, disablement, recovery, and deletion. KMS is valuable because it centralises and controls these operations.

### Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.

Envelope encryption creates a data-encryption key (DEK) to encrypt the large file locally. The DEK is then encrypted, or wrapped, by the KMS master key and stored with the ciphertext. An authorised client later asks KMS to unwrap the DEK and discards the plaintext after use.

Only the master key needs hardware-grade protection because it protects the small wrapped DEKs, not every byte of application data. This allows KMS to protect the master key in specialised infrastructure while local AES encryption remains efficient for large files.

### Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot in the cloud?

Cryptographic erasure destroys or disables the encryption key needed to decrypt data. Once tenant A's KMS key was disabled, its wrapped data key could no longer be decrypted, making the encrypted record unusable. This remains effective even if ciphertext exists in backups, replicas, snapshots, or storage media.

Cloud customers usually cannot prove that every physical copy of a file was overwritten because storage is virtualised and replicated. Provided no usable copy of the encryption key remains, key destruction makes the encrypted data computationally unrecoverable.

### Q5. How does a hash chain make a log tamper-evident?

Every entry in a hash chain contains a hash derived from its own content and the previous entry's hash. Changing an earlier entry changes its hash and breaks the link to the next entry and all later entries. Recomputing the chain therefore exposes alteration, insertion, or reordering. For stronger protection, the log should also use access-controlled or append-only storage and periodic external signing or anchoring.

## Verification Commands

The following commands verify that the KMS keys exist and that the RSA signature remains valid:

```powershell
aws --endpoint-url=http://localhost:4566 kms list-keys
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

![Verification command output](Evidence/8.%20Verification%20Command.png)

## Cleanup and Teardown

After capturing the evidence, the temporary TLS container, encryption artefacts, and LocalStack container can be removed:

```powershell
docker stop tls
Remove-Item record.*, private.pem, public.pem, key.pem, cert.pem, datakey.*, tampered.txt -ErrorAction SilentlyContinue
docker stop localstack
docker rm localstack
```

![Cleanup and teardown](Evidence/9.%20Cleanup%20%26%20Teardown.png)

## Security Best-Practices Checklist

- [x] Data was encrypted at rest with AES-256 and successful decryption was confirmed.
- [x] RSA keys were used correctly for public-key encryption and digital signatures.
- [x] Data was protected in transit with TLS.
- [x] Envelope encryption was used and the plaintext data key was removed from disk.
- [x] Separate tenant keys and cryptographic erasure were demonstrated.
- [x] File integrity and tamper-evident logging were verified with SHA-256 hashes.

## Conclusion

Lab 3 demonstrates that cloud data protection depends on layered cryptographic controls and secure key management. AES protects data at rest, RSA supports key distribution and signatures, and TLS protects data in transit. KMS enables centrally controlled envelope encryption, while per-tenant keys limit the impact of compromise and make cryptographic erasure possible. Finally, SHA-256 hashes and hash chains provide evidence that files and logs have not been altered.
