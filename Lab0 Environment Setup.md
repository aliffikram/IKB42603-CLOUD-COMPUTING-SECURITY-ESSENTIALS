# Lab 0: Environment Setup

**Course:** Cloud Computing Security Essentials  
**Lab:** 0 Environment Setup  
**Date:** 28 July 2026

## Objective

Prepare and verify a local Linux-based environment for the cloud-security laboratory exercises. The environment requires container tooling, Kubernetes utilities, AWS command-line access, and cryptographic/authentication tools.

## Environment

The verification was performed from an Ubuntu terminal (WSL/Linux shell). The following tools were checked:

| Tool | Purpose | Verified version |
| --- | --- | --- |
| Docker | Build and run containers | Docker 29.6.2, build dfc4efb |
| AWS CLI | Manage and interact with AWS services | aws-cli/2.36.9, Python/3.14.6, Linux/6.17.0-22-generic, exe/x86_64.ubuntu.24 |
| kind | Run local Kubernetes clusters using Docker | kind v0.23.0 |
| kubectl | Communicate with Kubernetes clusters | Client v1.36.3; Kustomize v5.8.1 |
| OpenSSL | Perform cryptographic and TLS-related operations | OpenSSL 3.0.13 (30 Jan 2024) |
| oathtool | Generate and validate one-time passwords | oathtool 2.6.11 |

## Verification Procedure and Evidence

### 1. Docker

The Docker installation was checked with:

```bash
docker --version
```

The command returned Docker version 29.6.2, confirming that the Docker command-line client is installed.

<img width="189" height="24" alt="Screenshot 2026-07-28 192156" src="https://github.com/user-attachments/assets/19bd3594-7e72-4d24-b1bd-b4bda51cf837" />


### 2. AWS Command Line Interface

The AWS CLI installation was checked with:

```bash
aws --version
```

The output confirms AWS CLI version 2.36.9 running with Python 3.14.6 on Ubuntu Linux.

<img width="370" height="23" alt="Screenshot 2026-07-28 192247" src="https://github.com/user-attachments/assets/b414a4d0-632b-4b94-b807-fede8dcfe9b4" />

### 3. kind

The Kubernetes-in-Docker utility was checked with:

```bash
kind --version
```

The command returned kind v0.23.0.

<img width="179" height="24" alt="Screenshot 2026-07-28 192307" src="https://github.com/user-attachments/assets/a5da56a2-270b-4466-8f65-f4bc3f6f9d80" />

### 4. kubectl

The Kubernetes client was checked with:

```bash
kubectl version --client
```

The installed client is v1.36.3 and includes Kustomize v5.8.1.

<img width="229" height="34" alt="Screenshot 2026-07-28 192410" src="https://github.com/user-attachments/assets/60d343b6-5410-497e-955d-d91948831902" />

### 5. Cryptographic and OTP utilities

OpenSSL and oathtool were verified using:

```bash
openssl version
oathtool --version
```

The environment includes OpenSSL 3.0.13 and oathtool 2.6.11. These tools will support later exercises involving encryption, certificates, hashes, and time-based one-time passwords.

<img width="395" height="112" alt="Screenshot 2026-07-28 192503" src="https://github.com/user-attachments/assets/99235849-b2cc-4e41-a569-f7da20171536" />

## Conclusion

The required command-line tools for the lab environment are installed and their versions have been recorded. Docker, AWS CLI, kind, kubectl, OpenSSL, and oathtool are available for subsequent cloud-computing security labs.
