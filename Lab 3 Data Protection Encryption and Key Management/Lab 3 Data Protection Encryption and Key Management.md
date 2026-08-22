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

Create a sample sensitive record
<img width="804" height="23" alt="1  Create a sample sensitive record" src="https://github.com/user-attachments/assets/3f7c46bf-bd27-432b-95bd-6ea865ff2c79" />


Encrypt the record with AES-256
<img width="906" height="70" alt="1 1  Encrypt with AES-256" src="https://github.com/user-attachments/assets/38e9f1f7-dbd5-48c7-997c-25c265a0a798" />

Encrypted ciphertext is unreadable

<img width="539" height="48" alt="1 2  Prove it is unreadable" src="https://github.com/user-attachments/assets/0323b9bd-8c8e-4532-a25e-8f62863368c5" />

Successful AES-256 decryption
<img width="915" height="87" alt="1 3  Decrypt back" src="https://github.com/user-attachments/assets/38d6cb0c-b2c6-43d3-9f39-3d5d5eb44aa5" />


AES provides fast confidentiality for stored data. However, the shared symmetric key must be protected because anyone who obtains it can decrypt the record.

### Task 2: Asymmetric Encryption and Digital Signatures

A 2048-bit RSA public/private key pair was generated. The public key encrypted the record, while the matching private key decrypted it. A SHA-256 signature was then created with the private key and successfully verified with the public key.

Generate the RSA key pair

<img width="575" height="67" alt="2  Generate a 2048-bit key pair" src="https://github.com/user-attachments/assets/3afd13df-2d05-4f84-b33a-742282cbf59c" />

Encrypt with the public key and decrypt with the private key
<img width="1021" height="43" alt="2 1  Encrypt with the PUBLIC key, decrypt with the PRIVATE key" src="https://github.com/user-attachments/assets/7c85038f-db09-4b69-88aa-f36df5f6d26e" />

Sign with the private key and verify with the public key
<img width="866" height="65" alt="2 2  Sign with the PRIVATE key; verify with the PUBLIC key" src="https://github.com/user-attachments/assets/3623f936-8722-4946-b880-ea8f8d564710" />

The encryption test demonstrates confidentiality without sharing the private key. The successful signature verification demonstrates integrity and confirms that the signature was created by the private-key holder.

### Task 3: Encryption in Transit with TLS

A self-signed certificate was generated and used by a small NGINX container serving HTTPS on port 8443. A `curl -k` request retrieved the record through the TLS-protected channel. The `-k` option was necessary only because the local certificate was self-signed and not trusted by a public certificate authority.

Generate a self-signed certificate
<img width="1201" height="308" alt="3  Generate a self-signed certificate" src="https://github.com/user-attachments/assets/de59ccb8-7998-4409-bcab-998a6e582632" />

Serve HTTPS from the container
<img width="654" height="110" alt="3 1  Serve HTTPS on port 8443 using a small container" src="https://github.com/user-attachments/assets/cee7fe9c-2fdc-463c-98b7-6ad2aa008190" />

Connect to the service over TLS
<img width="624" height="45" alt="3 2  Connect over TLS (-k accepts the self-signed cert)" src="https://github.com/user-attachments/assets/6fa6cbc8-d4e2-4ad2-b4d7-c632eb6a95cb" />

TLS protects the record while it travels over the network. Without TLS, an on-path attacker could read a plaintext HTTP request. In production, the server should use a trusted certificate and clients must validate the certificate rather than using `-k`.

## Session B: Key Management, Envelope Encryption and Integrity

### Task 4: Create and Use a KMS Master Key

A customer-managed KMS key for tenant A was created in LocalStack, and its KeyId was assigned to `KEY_A`. The KMS key was used to encrypt a small secret directly.

KMS master-key task
<img width="623" height="24" alt="4  Create and Use a KMS Master Key" src="https://github.com/user-attachments/assets/97f29f2a-9e58-4e60-bb27-d22ba5348608" />


Create tenant A's KMS key and capture its KeyId
<img width="644" height="289" alt="4 1  Create a customer master key (CMK) and capture its KeyId" src="https://github.com/user-attachments/assets/bf603ee0-5bfe-4f5a-83d6-ad4db873db7f" />


Assign the KeyId to KEY_A

<img width="639" height="21" alt="4 2  Copy the KeyId from the output into KEY_A below" src="https://github.com/user-attachments/assets/16a84ac7-33c6-4139-afc0-bce1abcda2da" />

Encrypt a small secret directly with KMS
<img width="1129" height="68" alt="4 3  Encrypt a small secret directly with KMS" src="https://github.com/user-attachments/assets/18ad99a9-a2ab-4a17-85a1-b523777a0c39" />


KMS centralises key lifecycle and access controls. Direct KMS encryption is suitable for small values; large application files should use envelope encryption.

### Task 5: Envelope Encryption

KMS generated an AES-256 data key in plaintext and encrypted forms. The plaintext data key was decoded and used locally to encrypt the record. It was then deleted from disk, leaving only the KMS-wrapped data key and the encrypted record.

Generate a plaintext and wrapped data key
<img width="876" height="134" alt="5  Ask KMS for a data key (returns plaintext + encrypted versions)" src="https://github.com/user-attachments/assets/4b545b07-82c0-45d2-81b6-c7ecd015e58b" />

Encrypt the record locally with the plaintext data key
<img width="693" height="90" alt="5 1  Encrypt the big file locally with the PLAINTEXT data key" src="https://github.com/user-attachments/assets/34f1e0a7-8e2f-4146-90e3-572361d365d4" />

Remove the plaintext data key from disk
<img width="597" height="65" alt="5 2  Destroy the plaintext data key from disk — keep only the wrapped copy" src="https://github.com/user-attachments/assets/d2d9d893-15db-489c-85ec-e09d73a0db47" />

Envelope encryption uses a fast local data-encryption key for the file while KMS protects the smaller data key. To decrypt later, an authorised client asks KMS to unwrap the stored encrypted data key, uses it briefly, and then discards it again.

### Task 6: Per-Tenant Keys and Cryptographic Erasure

A separate KMS key was created for tenant B and assigned to `KEY_B`, keeping it distinct from tenant A's key. Tenant A's key was scheduled for deletion and disabled immediately. After the key was disabled, an attempt to decrypt tenant A's wrapped data key failed.

Create a separate KMS key for tenant B
<img width="759" height="377" alt="6  A separate key for tenant B" src="https://github.com/user-attachments/assets/3d1f326e-0d3d-4c43-bf58-7c83b5309e65" />

Assign the KeyId to KEY_B

<img width="636" height="21" alt="6 1  Copy the KeyId from the output into KEY_B below" src="https://github.com/user-attachments/assets/d7576443-ab61-454d-88f6-e0e815d50e73" />

Schedule deletion and disable tenant A's key
<img width="607" height="122" alt="6 2  Schedule deletion of tenant A&#39;s key (min window)   Disable it immediately to simulate erasure" src="https://github.com/user-attachments/assets/cba0423f-6022-4fd3-ab6d-d3bdc7564d61" />

Decrypt fails after tenant A's key is disabled
<img width="1205" height="88" alt="6 3  Attempt to unwrap tenant A&#39;s data key now — it should FAIL" src="https://github.com/user-attachments/assets/8836cc47-2489-4a86-87db-940f2631bbcc" />

This proves cryptographic erasure: ciphertext and backups may remain, but without the KMS key they cannot be decrypted. Separate keys also reduce the impact of a key compromise to the affected tenant.

### Task 7: Integrity and Tamper-Evident Logging

A SHA-256 hash was calculated for the original record. After modifying a copy, its hash differed from the original hash. A simple hash chain was also generated, with every entry containing the previous hash.

Calculate the SHA-256 fingerprint
<img width="766" height="46" alt="7  Fingerprint the file" src="https://github.com/user-attachments/assets/d54dacb9-65a7-42eb-a97c-7bf64c3b43e2" />

Tampered copy produces a different hash
<img width="786" height="114" alt="7 1  Tamper with a copy and show the hash changes" src="https://github.com/user-attachments/assets/3fbaced8-7c14-4fea-ad16-286c3c1b88d2" />


Hash chain for a tamper-evident log
<img width="785" height="178" alt="7 2  Hash chain each entry includes the previous hash (tamper-evident log)" src="https://github.com/user-attachments/assets/0686eb48-8477-40d4-b501-0bf2b665eca2" />


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

Verification command output
<img width="765" height="131" alt="8  Verification Command" src="https://github.com/user-attachments/assets/09336ab3-0819-4fdb-bf83-878674328eee" />

## Cleanup and Teardown

After capturing the evidence, the temporary TLS container, encryption artefacts, and LocalStack container can be removed:

Cleanup and teardown
<img width="771" height="132" alt="9  Cleanup   Teardown" src="https://github.com/user-attachments/assets/5c097fad-6d47-4f8d-8061-84d9ec6044a9" />

## Security Best-Practices Checklist

- [x] Data was encrypted at rest with AES-256 and successful decryption was confirmed.
- [x] RSA keys were used correctly for public-key encryption and digital signatures.
- [x] Data was protected in transit with TLS.
- [x] Envelope encryption was used and the plaintext data key was removed from disk.
- [x] Separate tenant keys and cryptographic erasure were demonstrated.
- [x] File integrity and tamper-evident logging were verified with SHA-256 hashes.

## Conclusion

Lab 3 demonstrates that cloud data protection depends on layered cryptographic controls and secure key management. AES protects data at rest, RSA supports key distribution and signatures, and TLS protects data in transit. KMS enables centrally controlled envelope encryption, while per-tenant keys limit the impact of compromise and make cryptographic erasure possible. Finally, SHA-256 hashes and hash chains provide evidence that files and logs have not been altered.
