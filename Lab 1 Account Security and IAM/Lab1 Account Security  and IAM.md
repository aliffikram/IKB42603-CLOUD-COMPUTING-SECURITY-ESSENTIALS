# Lab 1: Account Security and IAM

**Course:** Cloud Computing Security Essentials  
**Lab:** 1 Account Security and IAM  
**Date:** 30 July 2026

## Objective
This lab introduces the core concepts of cloud account security, IAM identity design, and least-privilege access control. The purpose is to understand how cloud identities are created, how permissions are managed, and how Kubernetes RBAC enforces access boundaries in practice.

## Lab Overview
According to the lab manual, the exercise is divided into two sessions:

- Session A: cloud identity with LocalStack IAM
- Session B: enforced access control with Kubernetes RBAC

The lab uses Docker, LocalStack, AWS CLI, kind, and kubectl. The goal is to avoid real cloud usage and instead perform all identity and authorization testing locally using simulation tools.

## Environment
The lab uses a local container-based setup:

- Docker Desktop / Docker Engine for container runtime
- LocalStack to emulate AWS-compatible APIs
- AWS CLI pointed to `http://localhost:4566`
- kind and kubectl to create and test a Kubernetes cluster

## Session A: Cloud Identity with LocalStack

### One-Time Environment Setup
Verify Docker is running, then start LocalStack. Run each command in a terminal:

1. Confirm Docker is installed and running

<img width="188" height="23" alt="1" src="https://github.com/user-attachments/assets/e24ba10a-6839-4627-aa7e-624da2cdebe7" />

2. Start LocalStack (AWS-compatible cloud) in a container

<img width="599" height="68" alt="2" src="https://github.com/user-attachments/assets/87e6c0d6-29c5-40d1-b774-712692a3c57e" />

3. Confirm it is healthy (should list running services)

<img width="601" height="100" alt="3" src="https://github.com/user-attachments/assets/60ac9371-acd6-4011-b2b1-19db9d898bcb" />

Point the AWS CLI at LocalStack by creating a helper alias. On macOS/Linux use awslocal via a
shell function; on Windows PowerShell use the --endpoint-url flag shown throughout.

Configure dummy credentials (LocalStack accepts any value)

<img width="308" height="32" alt="4" src="https://github.com/user-attachments/assets/3723672a-4f3a-4351-9979-885a9354fc08" />

Test: this talks to LocalStack, NOT real AWS

<img width="431" height="67" alt="5" src="https://github.com/user-attachments/assets/ba6f8ada-c39a-477b-97d8-88560f915994" />

### Task 1: Map the Cloud Identity Landscape
Before creating identities, it is important to understand the meaning of each cloud identity object.

| Concept | AWS term | Purpose |
| --- | --- | --- |
| All-powerful owner | Root user | The account with unrestricted access; should not be used for regular daily operations. |
| Human or application identity | IAM User | An identity that represents a person or service that can authenticate to AWS/LocalStack. |
| Permission bundle | IAM Policy | A document that defines what actions are allowed or denied. |
| Collection of users | IAM Group | A logical grouping used to manage permissions for multiple users more efficiently. |
| Temporary identity | IAM Role | A trusted identity that can be assumed temporarily and is commonly used for least-privilege access. |

### Task 2: Create a Least-Privilege Admin
The root account is a liability. The safer model is to create a dedicated admin identity and assign permissions through a group.

Example commands used in the lab:

<img width="312" height="13" alt="6" src="https://github.com/user-attachments/assets/e400639a-af28-4093-ab8f-73b804fced34" />

2.1Create a group and attach an admin policy to the GROUP

<img width="326" height="133" alt="7" src="https://github.com/user-attachments/assets/3c894390-7ac9-43b9-a15d-7f5108cef5bc" />

2.2 Create your personal admin user (replace YOURNAME)

<img width="392" height="111" alt="8" src="https://github.com/user-attachments/assets/9f047891-a2d3-4bd7-92fe-732b38805ffb" />

2.3 Put the user in the group (permissions flow from the group)

<img width="365" height="23" alt="9" src="https://github.com/user-attachments/assets/2a33f846-6804-49b8-b99e-d33c388de498" />

2.4 Verify the membership

<img width="371" height="211" alt="10" src="https://github.com/user-attachments/assets/6c83e5de-6471-4030-b32d-ca4988ecfe41" />

This practice is preferred because permissions are managed centrally through groups instead of being attached directly to individual users. It improves maintainability and makes auditing simpler.

### Task 3: Enforce Least Privilege with a Scoped Policy
A read-only identity was created to demonstrate reduced blast radius.

3.1 Create a read-only user

<img width="265" height="81" alt="12" src="https://github.com/user-attachments/assets/cd405297-cc30-4c7a-8705-9b6fe1083a52" />

3.2 Attach a scoped, read-only policy (S3 read only)

<img width="601" height="23" alt="13" src="https://github.com/user-attachments/assets/82162f02-88e7-47b0-af88-8fbd11e6bce8" />

3.3 List what the user can do

<img width="461" height="99" alt="14" src="https://github.com/user-attachments/assets/72704664-b13c-4787-9ec6-43c06c71077f" />

If the Analyst account were stolen, the damage would be limited compared to an admin account because the analyst only has read-only access. This reduces the blast radius and prevents the attacker from modifying or deleting important resources.

### Task 4: Credential Hygiene and Access Keys
Programmatic access often uses access keys, but long-lived keys create risk if they are leaked or stored insecurely.

4.1 Create an access key for the Analyst

<img width="410" height="110" alt="15" src="https://github.com/user-attachments/assets/a6705e0f-9d6d-4a4e-8423-cf24e38534bf" />

4.2 List access keys (note the AccessKeyId and status)

<img width="404" height="124" alt="16" src="https://github.com/user-attachments/assets/73dd459d-e038-453f-ac8b-0ecfb2694530" />

4.3 Rotate: deactivate the old key (paste the AccessKeyId)

<img width="420" height="23" alt="17" src="https://github.com/user-attachments/assets/c597ad51-4c46-457c-9a7e-a5f754b7da52" />

This demonstrates credential rotation and deactivation. In real cloud environments, root users should never use access keys, and keys should never be committed to public repositories. Roles are preferred over permanent access keys when possible.

## Session B: Enforced Access Control with Kubernetes RBAC

### Setup Create a Local Kubernetes Cluster

Create a throwaway cluster (runs inside Docker)

<img width="440" height="157" alt="18" src="https://github.com/user-attachments/assets/6e8e9803-f0ec-4f13-b7f2-355ea10abe22" />

Confirm it is up

<img width="522" height="89" alt="19" src="https://github.com/user-attachments/assets/bf992aeb-d8dc-4fad-96be-3c7ae3d5437c" />

### Task 5: Separate Environments with Namespaces

<img width="249" height="143" alt="20" src="https://github.com/user-attachments/assets/915d8e4c-1a1a-4470-bb60-1d641957302b" />

### Task 6: Define a Role and Bind It
A role is created that allows limited read access to pods in the `dev` namespace, and it is bound to a service account.

6.1 Create a service account to represent a developer

<img width="333" height="22" alt="21" src="https://github.com/user-attachments/assets/b63abbcc-f214-4447-b933-d5ca59b2914e" />

6.2 Create a Role that allows only get/list/watch on pods in dev

<img width="304" height="34" alt="22" src="https://github.com/user-attachments/assets/860b5e60-d267-4947-8707-1e95fab06ad1" />

6.3 Bind the Role to the service account

<img width="367" height="32" alt="23" src="https://github.com/user-attachments/assets/6218c62c-3265-4b20-a4cc-bf35211c8e58" />

Here, the role defines permissions, while the role binding maps those permissions to the identity.

### Task 7: Test That Access Control Works
The following authorization checks are used to prove the role is enforced:

SA=system:serviceaccount:dev:dev-user

Should be YES — reading pods in dev is allowed

<img width="329" height="24" alt="24  7 1 Should be YES — reading pods in dev is allowed" src="https://github.com/user-attachments/assets/53153519-89cd-4be0-862b-8ac5c8452003" />

Should be NO — deleting pods is not granted

<img width="340" height="23" alt="25" src="https://github.com/user-attachments/assets/60575e32-e555-47d0-8433-753c0955eb36" />

Should be NO — the role does not extend to prod

<img width="332" height="23" alt="26" src="https://github.com/user-attachments/assets/bbd96360-b595-4020-982e-d213eefbef23" />

Expected results:
- `YES` for listing pods in `dev`
- `NO` for deleting pods in `dev`
- `NO` for listing pods in `prod`

This demonstrates the principle of least privilege: the developer service account can only perform the actions allowed by the role and nothing more.

## Short-Answer Questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?
Attaching policies to groups is better because it centralizes permission management, simplifies administration, and improves auditability. When you update the group policy, all members automatically receive the changed permissions.

### Q2. What is the difference between an IAM User and an IAM Role?
An IAM User is a long-lived identity used by people or services to authenticate directly. An IAM Role is a temporary identity that is assumed when needed and is commonly used for secure access delegation and least-privilege practices.

### Q3. Explain least privilege using the Analyst account and how it reduces blast radius if compromised.
The Analyst account is limited to read-only access. If its credentials are stolen, an attacker can only view data rather than modify or delete resources. This greatly reduces the blast radius because the impact is contained.

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?
A Role defines what actions are allowed on which resources. A RoleBinding connects that role to a specific identity, such as a service account, user, or group.

### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?
The service account failed to access `prod` because the role and role binding were scoped only to `dev`. This demonstrates the security principle of least privilege and namespace isolation.

## Verification Command
To prove the RBAC configuration is in place, the following command was used:

<img width="385" height="188" alt="27" src="https://github.com/user-attachments/assets/7fbc35b9-ddc2-4581-960d-ed20527332fd" />

The output should show the binding that maps the `pod-reader` role to the `dev-user` service account.

## Cleanup & Teardown

Remove the Kubernetes cluster

<img width="287" height="34" alt="28" src="https://github.com/user-attachments/assets/c2b60dd6-6f45-4f93-8bb1-e68b5cbd3526" />

## Security Best-Practices Checklist
- [x] Root user is not used for daily tasks.
- [x] Permissions are granted through groups and roles.
- [x] A least-privilege read-only identity was created and tested.
- [x] Access keys were listed and rotated/deactivated.
- [x] Kubernetes RBAC blocked an unauthorized action.

## Conclusion
Lab 1 shows that account security and IAM are essential elements of cloud security. By using dedicated identities, group-based permissions, least-privilege policies, and Kubernetes RBAC, organizations can reduce their attack surface and enforce secure access control. The lab demonstrates that effective identity governance is not only about authentication, but also about precise authorization and continuous verification of access boundaries.
