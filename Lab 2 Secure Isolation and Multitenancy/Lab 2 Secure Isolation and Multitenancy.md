# Lab 2: Secure Isolation and Multi-Tenancy

**Course:** Cloud Computing Security Essentials  
**Lab:** 2 Secure Isolation and Multi-Tenancy  
**Date:** 7 August 2026

## Objective

This lab demonstrates how to isolate multiple tenants sharing Kubernetes and Docker infrastructure. The objectives are to separate tenants at the compute layer, identify the default-open network risk, enforce network isolation with a default-deny policy, protect tenant secrets through RBAC, and explain data remanence and secure deletion.

## Lab Overview

The lab is divided into two sessions:

- **Session A:** compute isolation, Kubernetes namespaces, the default-open network risk, and resource quotas.
- **Session B:** default-deny network isolation, tenant secret isolation, and data remanence.

The environment uses Docker, kind, Kubernetes, Calico NetworkPolicy enforcement, and `kubectl`.

## Environment Setup

A kind cluster named `ccse-lab2` was created with the default CNI disabled. Calico was then installed so Kubernetes `NetworkPolicy` rules would be enforced.

![Cluster created with NetworkPolicy enforcement](Evidence/0.1%20Setup-Cluster%20with%20Policy%20Enforcement.png)

![Calico policy enforcement setup](Evidence/0.2%20Setup-Cluster%20with%20Policy%20Enforcement%202.png)

## Session A: Compute Isolation and the Default-Open Risk

### Task 1: Two Tenants on One Cluster

Two namespaces, `tenant-a` and `tenant-b`, model separate customers sharing the same Kubernetes cluster. An NGINX web deployment and service were created in each namespace.

![Create tenant namespaces](Evidence/1.1%20Model%20two%20customers%20as%20two%20namespaces%20sharing%20the%20same%20physical%20infrastructure..png)

![Deploy a web server for each tenant](Evidence/1.2%20Create%20deploy%20a%20simple%20web%20server%20for%20each%20tenant.png)

![Expose each web deployment as a service](Evidence/1.3%20Expose%20deploy%20a%20simple%20web%20server%20for%20each%20tenant.png)

![Pods and services running in both tenant namespaces](Evidence/1.4%20Showing%20pods%20and%20services%20in%20tenant-a%20and%20tenant-b%20running.png)

Namespaces provide a logical compute and management boundary, but they do not automatically block network traffic between tenants.

### Task 2: Observe the Default-Open Risk

The ClusterIP address of `tenant-b`'s web service was obtained. A temporary curl pod in `tenant-a` then accessed that address and received `HTTP 200`.

![Tenant-b service IP](Evidence/2.1%20Get%20tenant-b's%20service%20IP.png)

![Cross-tenant probe returns HTTP 200 before isolation](Evidence/2.2%20From%20tenant-a%2C%20curl%20tenant-b%20%28replace%20B_IP%29.png)

This proves the default-open risk: a workload in one tenant could reach a workload in another tenant unless a network policy is configured.

### Task 3: Contain the Noisy Neighbour

A `ResourceQuota` was applied to `tenant-a` to limit requests to 1 CPU, 512 MiB memory, and five pods.

![Resource quota applied to tenant-a](Evidence/3.1%20Contain%20the%20Noisy%20Neighbour%20%28Resource%20Quotas%29.png)

The quota prevents a single tenant from consuming an uncontrolled amount of shared cluster capacity.

## Session B: Network and Storage Isolation

### Task 4: Default-Deny Network Isolation

A `default-deny-ingress` `NetworkPolicy` was applied to all pods in `tenant-b`.

![Default-deny ingress policy for tenant-b](Evidence/4.1%20Deny%20ALL%20ingress%20into%20tenant-b.png)

When the same probe was run from `tenant-a`, it timed out or failed rather than returning `HTTP 200`.

![Cross-tenant probe fails after policy enforcement](Evidence/4.2%20Re-run%20the%20SAME%20probe%20from%20Task%202-it%20should%20now%20TIME%20OUT%20%20fail.png)

The before-and-after result confirms that the NetworkPolicy enforces tenant network segmentation.

### Task 5: Storage and Secret Isolation

A separate secret was created in each tenant namespace. The `app-a` service account was given a Role and RoleBinding that permit reading secrets only in `tenant-a`.

![A secret created for each tenant](Evidence/5.1%20Create%20a%20secret%20in%20each%20tenant.png)

![Service account and tenant-a-scoped RBAC](Evidence/5.2%20A%20service%20account%20scoped%20to%20tenant-a%20only.png)

The authorization checks show that the service account can read secrets in `tenant-a` but cannot read secrets in `tenant-b`.

![Authorization results for tenant-a and tenant-b](Evidence/5.3%20Both%20authorization%20outputs-tenant%20a%20%26%20tenant%20b.png)

This is storage isolation enforced through namespace-scoped Kubernetes RBAC and least privilege.

### Task 6: Data Remanence and Secure Deletion

The first test created a sensitive file in a Docker volume and removed it normally, demonstrating that deletion alone may not securely remove underlying data.

![Normal deletion and remanence scan](Evidence/6.1%20Create%20a%20file%2C%20delete%20it%20normally%2C%20then%20show%20the%20bytes%20may%20persist.png)

The second test overwrote the file with zeros before deleting it.

![Overwrite before deletion using shred](Evidence/6.2%20Secure%20wipe%20overwrite%20before%20delete%20%28shred%29.png)

In cloud environments, physical storage blocks are usually not directly controlled by the tenant. Therefore, cryptographic erasure—destroying the encryption key—is generally the preferred method for making data unrecoverable.

## Short-Answer Questions

### Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?

Namespaces separate Kubernetes resources logically, but Kubernetes networking is normally flat and allows pod-to-pod communication across namespaces unless policies restrict it. This is dangerous because one tenant could probe, access, or attack another tenant's exposed services.

### Q2. Explain the default-deny principle and how your NetworkPolicy implements it.

Default-deny means traffic is blocked unless an explicit rule permits it. The `default-deny-ingress` policy selects every pod in `tenant-b` and declares ingress policy without any allow rules, so inbound traffic—including the probe from `tenant-a`—is denied.

### Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?

Containers share the host kernel, so their isolation depends on the container runtime and kernel security controls. Virtual machines have separate guest operating systems and kernels, providing a stronger boundary. A VM boundary should be added for highly sensitive workloads, untrusted tenants, strict compliance requirements, or when the impact of a container escape is unacceptable.

### Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?

Data remanence is residual data that may remain recoverable after a file is deleted. Secure overwriting is not always reliable or controllable on cloud storage because storage is virtualized and may be replicated. Cryptographic erasure removes the encryption key, making all data encrypted with that key unreadable without needing access to the physical media.

### Q5. Which of the three isolation dimensions—compute, network, and storage—did each task exercise?

| Task | Isolation dimension | Control demonstrated |
| --- | --- | --- |
| Task 1 | Compute | Separate tenant namespaces and workloads on one cluster. |
| Task 2 | Network | Default-open cross-namespace connectivity risk. |
| Task 3 | Compute | ResourceQuota limits noisy-neighbour resource consumption. |
| Task 4 | Network | Default-deny ingress NetworkPolicy blocks cross-tenant traffic. |
| Task 5 | Storage | Per-tenant secrets protected by namespace-scoped RBAC. |
| Task 6 | Storage | Data remanence awareness and secure wipe/cryptographic erasure. |

## Verification Commands

The following commands verify the two main cluster controls:

```powershell
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

![Verification command output](Evidence/7.1%20Verification%20Command.png)

## Cleanup and Teardown

After capturing the evidence, the temporary Kubernetes cluster and Docker volume can be removed:

```powershell
kind delete cluster --name ccse-lab2
docker volume rm ccse-vol
```

![Cleanup and teardown](Evidence/8.1%20Cleanup%20%26%20Teardown.png)

## Security Best-Practices Checklist

- [x] Tenants are separated into distinct Kubernetes namespaces.
- [x] A ResourceQuota limits tenant-a's use of shared compute capacity.
- [x] A default-deny NetworkPolicy blocks cross-tenant ingress traffic.
- [x] Tenant secrets are protected with namespace-scoped RBAC.
- [x] Data remanence and secure deletion/cryptographic erasure are understood.

## Conclusion

Lab 2 shows that multi-tenancy requires multiple, complementary isolation controls. Namespaces and quotas isolate workload management and capacity, NetworkPolicy enforces traffic segmentation, RBAC protects tenant secrets, and secure deletion practices reduce storage-remanence risk. Together, these controls reduce the blast radius of a compromised or noisy tenant on shared cloud infrastructure.
