# Quiz Week 2: Cloud Computing Security Essentials

**Course:** Cloud Computing Security Essentials  
**Assessment:** Quiz Week 2  
**Date:** 4 August 2026

## Objective

Review core cloud-computing, AWS IAM, Docker, Kubernetes, and LocalStack concepts, and provide the verified answer for every quiz question.

## Answer Review and Evidence

Each section includes the original quiz evidence, the correct answer, and a short explanation.

### Question 1: Which service model provides virtual machines?

<img width="477" height="185" alt="Question 1" src="https://github.com/user-attachments/assets/45a0099b-6ca9-43b3-887a-c0a6988f4666" />

**Correct answer: IaaS**

Infrastructure as a Service (IaaS) provides virtualized infrastructure, including virtual machines, storage, and networking.

### Question 2: A collection of IAM users is called:

<img width="476" height="187" alt="Question 2" src="https://github.com/user-attachments/assets/c5d49844-f45a-4ef9-87c1-52ddd1263fb4" />

**Correct answer: IAM Group**

An IAM group is a collection of IAM users used to manage permissions for them together.

### Question 3: Which IAM identity is normally used as a temporary identity?

<img width="476" height="185" alt="Question 3" src="https://github.com/user-attachments/assets/cdcc5bff-756d-46f1-8f7a-9fed03afb55e" />

**Correct answer: IAM Role**

IAM roles provide temporary credentials when assumed by a user, service, or workload.

### Question 4: If an access key is compromised, what should be done first?

<img width="477" height="185" alt="Question 4" src="https://github.com/user-attachments/assets/7ad5e976-63ea-4812-88dd-24d96e1611e5" />

**Correct answer: Deactivate or rotate the key**

Immediately disable the exposed key or rotate it to stop unauthorized use while replacing it safely.

### Question 5: Which is NOT an essential characteristic of cloud computing?

<img width="426" height="185" alt="Question 5" src="https://github.com/user-attachments/assets/a1689ea2-c315-4822-aea0-7d65f7012216" />

**Correct answer: Manual Provisioning**

Cloud computing relies on on-demand self-service; manual provisioning is not an essential cloud characteristic.

### Question 6: Google Docs is an example of:

<img width="476" height="185" alt="Question 6" src="https://github.com/user-attachments/assets/d73cafba-e043-4f29-9d80-86725c380d87" />

**Correct answer: SaaS**

Google Docs is a ready-to-use application delivered through the internet, which makes it Software as a Service (SaaS).

### Question 7: Which account should never have access keys created for routine use?

<img width="478" height="185" alt="Question 7" src="https://github.com/user-attachments/assets/51664669-6d7b-4a3e-89a5-7c2ef3f859f5" />

**Correct answer: Root User**

Root-user access keys should not be used for routine work because the root user has unrestricted account access.

### Question 8: Which IAM component contains permissions?

<img width="477" height="185" alt="Question 8" src="https://github.com/user-attachments/assets/f71f41da-d7b1-4a52-b13b-a817b80253e7" />

**Correct answer: IAM Policy**

An IAM policy is the document that defines permissions, including allowed or denied actions on resources.

### Question 9: Cloud computing refers to:

<img width="476" height="186" alt="Question 9" src="https://github.com/user-attachments/assets/027965fe-6b0d-44d0-83e7-83d29deafd57" />

**Correct answer: Delivering computing resources over the Internet**

Cloud computing provides on-demand computing services through a network, commonly the internet.

### Question 10: Which deployment model provides the MOST control?

<img width="476" height="184" alt="Question 10" src="https://github.com/user-attachments/assets/0ff1ddff-61b6-4ca7-9c0d-603557603bca" />

**Correct answer: Private Cloud**

A private cloud is dedicated to one organization and generally provides the greatest control over infrastructure and security.

### Question 11: In the ARN `arn:aws:s3:::my-bucket`, which component represents the AWS service?

<img width="478" height="200" alt="Question 11" src="https://github.com/user-attachments/assets/9eb4374f-1ac2-4828-83d5-4dd0dd4f30c8" />

**Correct answer: s3**

In this ARN, `s3` is the service namespace.

### Question 12: Which AWS identity has unlimited privileges?

<img width="475" height="185" alt="Question 12" src="https://github.com/user-attachments/assets/31f757ca-8fef-40ba-98c9-53dc590f0979" />

**Correct answer: Root User**

The AWS account root user begins with full, unrestricted access to the account.

### Question 13: Which characteristic allows cloud resources to automatically grow or shrink?

<img width="476" height="202" alt="Question 13" src="https://github.com/user-attachments/assets/6b1ae071-ac69-4887-96c5-213cb282149f" />

**Correct answer: Rapid Elasticity**

Rapid elasticity lets resources scale out or in automatically as demand changes.

### Question 14: What does ARN stand for?

<img width="429" height="185" alt="Question 14" src="https://github.com/user-attachments/assets/40b60c78-31f4-4949-8127-3ff44d998449" />

**Correct answer: Amazon Resource Name**

ARN stands for Amazon Resource Name, AWS's standard identifier format for resources.

### Question 15: Which command lists Kubernetes nodes?

<img width="473" height="187" alt="Question 15" src="https://github.com/user-attachments/assets/edbebcfa-a7dd-4273-afda-a4423753c577" />

**Correct answer: `kubectl get nodes`**

This Kubernetes CLI command lists the nodes registered in the cluster.

### Question 16: A Kubernetes cluster consists of:

<img width="473" height="185" alt="Question 16" src="https://github.com/user-attachments/assets/e89f14d0-eb7b-4e51-af71-75d9ac0cf915" />

**Correct answer: Multiple nodes**

A Kubernetes cluster comprises one or more nodes. Of the options shown, "Multiple nodes" is the best answer.

### Question 17: LocalStack is used because it:

<img width="475" height="185" alt="Question 17" src="https://github.com/user-attachments/assets/4af6f2c6-1c91-47e3-a044-270944c1557b" />

**Correct answer: Simulates AWS services locally**

LocalStack emulates many AWS services on a local machine for development and testing.

### Question 18: A node is:

<img width="475" height="185" alt="Question 18" src="https://github.com/user-attachments/assets/81cb2d4a-879e-40c4-914c-aa15af0be5d9" />

**Correct answer: A worker machine**

A Kubernetes node is a worker machine, physical or virtual, that runs pods.

### Question 19: Which service model requires customers to manage the operating system?

<img width="475" height="184" alt="Question 19" src="https://github.com/user-attachments/assets/b7f32094-1df6-4ebf-b358-ca2a1b7edf27" />

**Correct answer: IaaS**

With Infrastructure as a Service, the provider manages the physical infrastructure while the customer manages the operating system and above.

### Question 20: Which deployment model combines private and public cloud?

<img width="473" height="185" alt="Question 20" src="https://github.com/user-attachments/assets/977073ab-f515-4eda-b0ec-65e7529e66ff" />

**Correct answer: Hybrid Cloud**

A hybrid cloud combines private-cloud and public-cloud environments.

### Question 21: Which AWS CLI command verifies the current identity?

<img width="475" height="185" alt="Question 21" src="https://github.com/user-attachments/assets/e58ba8dd-9d4d-439c-9d7d-5f74b5117058" />

**Correct answer: `aws sts get-caller-identity`**

This AWS CLI command returns the account and IAM principal for the current credentials.

### Question 22: The smallest deployable unit in Kubernetes is:

<img width="476" height="185" alt="Question 22" src="https://github.com/user-attachments/assets/a2f1c520-d383-4ec1-9d92-a15d8759a941" />

**Correct answer: Pod**

A pod is Kubernetes's smallest deployable unit and can contain one or more containers.

### Question 23: Which ARN component identifies the AWS account that owns the resource?

<img width="473" height="184" alt="Question 23" src="https://github.com/user-attachments/assets/148f6dbd-1263-4368-b9e3-d186b942a328" />

**Correct answer: Account ID**

The account-ID field in an ARN identifies the AWS account that owns the resource.

### Question 24: Docker is mainly used to:

<img width="474" height="182" alt="Question 24" src="https://github.com/user-attachments/assets/276c17d5-76c1-464b-9918-164af32236dd" />

**Correct answer: Run containers**

Docker packages and runs applications in containers.

### Question 25: Which AWS-managed policy provides full administrative access?

<img width="473" height="184" alt="Question 25" src="https://github.com/user-attachments/assets/085bf8ca-fc7c-4f0f-b3e2-e3b6900d959a" />

**Correct answer: AdministratorAccess**

`AdministratorAccess` is the AWS-managed policy that grants full access to AWS services and resources.

### Question 26: Which tool creates a local Kubernetes cluster?

<img width="472" height="185" alt="Question 26" src="https://github.com/user-attachments/assets/e6fbfc48-bdb8-4df0-a21a-8d40e38c5f97" />

**Correct answer: kind**

kind (Kubernetes IN Docker) creates local Kubernetes clusters using Docker containers as nodes.

### Question 27: Access keys are mainly used for:

<img width="476" height="184" alt="Question 27" src="https://github.com/user-attachments/assets/c42f5595-8c9e-4c43-8b82-fc110eb3a853" />

**Correct answer: Programmatic access**

Access keys authenticate programmatic requests, such as requests through the AWS CLI, SDKs, or APIs.

### Question 28: For easier permission management, policies should preferably be attached to:

<img width="473" height="200" alt="Question 28" src="https://github.com/user-attachments/assets/a651c7ff-3068-4d53-92db-39620a57dac3" />

**Correct answer: IAM Groups**

Attaching policies to groups makes permissions easier to manage consistently for multiple users.

### Question 29: Which security principle gives users only the permissions required to perform their tasks?

<img width="473" height="203" alt="Question 29" src="https://github.com/user-attachments/assets/30c1d9f7-abe3-4251-895e-c72019bebbe8" />

**Correct answer: Principle of Least Privilege**

Least privilege grants only the minimum permissions needed to complete assigned tasks.

### Question 30: Which endpoint is commonly used with LocalStack?

<img width="474" height="185" alt="Question 30" src="https://github.com/user-attachments/assets/aefb56b7-3521-4f70-859e-019bbbc90db7" />

**Correct answer: `http://localhost:4566`**

LocalStack commonly exposes its unified edge endpoint on port `4566`.

## Conclusion

This answer review covers the essential concepts assessed in Quiz Week 2: cloud service and deployment models, AWS IAM and ARN fundamentals, credential hygiene, containerization, Kubernetes, and LocalStack. Each answer has been verified against the available options and supported with the original quiz evidence.
