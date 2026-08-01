# Lab 1: Account Security and IAM

**Course:** Cloud Computing Security Essentials  
**Lab:** 1 — Account Security and IAM  
**Date:** 1 August 2026

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

## Session A — Cloud Identity with LocalStack

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

```bash
EP='--endpoint-url=http://localhost:4566'
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
    --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws $EP iam create-user --user-name CloudAdmin_YOURNAME
aws $EP iam add-user-to-group --group-name Admins \
    --user-name CloudAdmin_YOURNAME
aws $EP iam get-group --group-name Admins
```

This practice is preferred because permissions are managed centrally through groups instead of being attached directly to individual users. It improves maintainability and makes auditing simpler.

### Task 3: Enforce Least Privilege with a Scoped Policy
A read-only identity was created to demonstrate reduced blast radius.

Example commands:

```bash
aws $EP iam create-user --user-name Analyst_YOURNAME
aws $EP iam attach-user-policy --user-name Analyst_YOURNAME \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws $EP iam list-attached-user-policies --user-name Analyst_YOURNAME
```

If the Analyst account were stolen, the damage would be limited compared to an admin account because the analyst only has read-only access. This reduces the blast radius and prevents the attacker from modifying or deleting important resources.

### Task 4: Credential Hygiene and Access Keys
Programmatic access often uses access keys, but long-lived keys create risk if they are leaked or stored insecurely.

Example commands:

```bash
aws $EP iam create-access-key --user-name Analyst_YOURNAME
aws $EP iam list-access-keys --user-name Analyst_YOURNAME
aws $EP iam update-access-key --user-name Analyst_YOURNAME \
    --access-key-id <PASTE_KEY_ID> --status Inactive
```

This demonstrates credential rotation and deactivation. In real cloud environments, root users should never use access keys, and keys should never be committed to public repositories. Roles are preferred over permanent access keys when possible.

## Verification Evidence

### Screenshot 1 — Identity Verification
The identity used by the local AWS CLI session was verified through the STS command:

```bash
aws --endpoint-url=http://localhost:4566 sts get-caller-identity
```

This output proves the operating identity and confirms that the session is communicating with LocalStack instead of a real AWS account.

![Lab 1 account security overview](1.png)

## Session B — Enforced Access Control with Kubernetes RBAC

### Task 5: Separate Environments with Namespaces
The lab creates separate namespaces to represent different environments:

```bash
kind create cluster --name ccse-lab1
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

This is important because RBAC can be scoped to namespace boundaries as well as cluster-wide boundaries.

### Task 6: Define a Role and Bind It
A role is created that allows limited read access to pods in the `dev` namespace, and it is bound to a service account.

```bash
kubectl create serviceaccount dev-user -n dev
kubectl create role pod-reader -n dev \
    --verb=get,list,watch --resource=pods
kubectl create rolebinding dev-user-binding -n dev \
    --role=pod-reader --serviceaccount=dev:dev-user
```

Here, the role defines permissions, while the role binding maps those permissions to the identity.

### Task 7: Test That Access Control Works
The following authorization checks are used to prove the role is enforced:

```bash
SA=system:serviceaccount:dev:dev-user
kubectl auth can-i list pods -n dev --as=$SA
kubectl auth can-i delete pods -n dev --as=$SA
kubectl auth can-i list pods -n prod --as=$SA
```

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

```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

The output should show the binding that maps the `pod-reader` role to the `dev-user` service account.

## Security Best-Practices Checklist
- [x] Root user is not used for daily tasks.
- [x] Permissions are granted through groups and roles.
- [x] A least-privilege read-only identity was created and tested.
- [x] Access keys were listed and rotated/deactivated.
- [x] Kubernetes RBAC blocked an unauthorized action.

## Conclusion
Lab 1 shows that account security and IAM are essential elements of cloud security. By using dedicated identities, group-based permissions, least-privilege policies, and Kubernetes RBAC, organizations can reduce their attack surface and enforce secure access control. The lab demonstrates that effective identity governance is not only about authentication, but also about precise authorization and continuous verification of access boundaries.
