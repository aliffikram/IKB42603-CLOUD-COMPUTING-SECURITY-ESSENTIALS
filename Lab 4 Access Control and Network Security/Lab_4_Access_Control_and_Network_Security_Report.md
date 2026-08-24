# Lab 4: Access Control and Network Security

| Item | Details |
|---|---|
| Course | Cloud Computing Security Essentials |
| Lab | Lab 4 — Access Control and Network Security |
| Student | ______________________________ |
| Student ID | ______________________________ |
| Date performed | 24 August 2026 |

## Objective

To apply and verify practical access-control and network-security controls in containerised and Kubernetes environments. The lab covers HTTP Basic Authentication, time-based one-time passwords (TOTP), Kubernetes role-based access control (RBAC), Docker network segmentation, firewall default-deny rules, container hardening, vulnerability scanning, verification, and cleanup.

## Environment and tools

The activities were performed in an Ubuntu terminal using Docker, Nginx, `curl`, `oathtool`, Kind/Kubernetes, `kubectl`, `iptables`, and Trivy.

## Procedure, results, and evidence

### 0. Prerequisites

Docker and the TOTP utility were installed before the security exercises were started.

![Evidence 0 — Installation command](<Evidence/0. Install Docker oathtool.png>)

*Figure 0. Docker/oathtool prerequisite installation.*

![Evidence 0.1 — Installation output](<Evidence/0.1. Install Docker oathtool.png>)

*Figure 0.1. Ubuntu package update and installation output.*

### 1. HTTP Basic Authentication

An Nginx password file was created for the user `student`, an Nginx server configuration was generated, and the authentication service was started in a container.

![Evidence 1 — Password file](<Evidence/1. Create a password file (user student).png>)

*Figure 1. Password file creation for the `student` account.*

![Evidence 1.1 — Nginx configuration](<Evidence/1.1. Generate the Nginx server configuration.png>)

*Figure 1.1. Nginx authentication server configuration.*

![Evidence 1.2 — Nginx container](<Evidence/1.2. Run the Nginx authentication service container.png>)

*Figure 1.2. Starting the Nginx authentication service container.*

The supplied test output shows a `200` response for the unauthenticated request and `Authenticated OK` when credentials for `student` were submitted. The `200` response should be reviewed if the intended policy is to deny anonymous access; a correctly protected endpoint normally returns `401 Unauthorized` without credentials.

![Evidence 1.3 — Access test](<Evidence/1.3. Test unauthenticated vs authenticated access.png>)

*Figure 1.3. Unauthenticated and authenticated HTTP access test.*

### 2. Multi-factor authentication using TOTP

A Base32 shared secret was used to generate a current six-digit TOTP value. The entered code was compared against the expected value, and the captured validation returned `MFA OK`.

![Evidence 2 — Generate TOTP](<Evidence/2. Create a shared secret (base32) and generate the current 6-digit code.png>)

*Figure 2. TOTP shared secret and current-code generation.*

![Evidence 2.1 — Validate TOTP](<Evidence/2.1. Validate a code the user types (compare to the expected value).png>)

*Figure 2.1. Successful TOTP validation (`MFA OK`).*

### 3. Kubernetes RBAC

A Kind cluster and application resources were created. A developer service account was assigned a role permitting read access to pods in the `app` namespace. Permission checks confirmed that the account may list pods but may not create or delete them.

![Evidence 3 — Kind cluster and resources](<Evidence/3. Create a Kind cluster and resources.png>)

*Figure 3. Creating the Kind cluster and Kubernetes resources.*

![Evidence 3.1 — Read-only pod access](<Evidence/3.1. Developer may only read pods.png>)

*Figure 3.1. Developer service account read-only pod access.*

![Evidence 3.2 — RBAC permission checks](<Evidence/3.2. Test role permissions.png>)

*Figure 3.2. `kubectl auth can-i` results: list = yes; create/delete = no.*

![Evidence 3.3 — Authentication container stopped](<Evidence/3.3. Stop the authentication container.png>)

*Figure 3.3. Authentication container stopped after the exercise.*

### 4. Docker network segmentation

Two Docker networks, `frontend-net` and `backend-net`, were created. The database container was connected only to the backend network, the application container to both networks, and the web container only to the frontend network. The connectivity test showed that `web` could not resolve/reach `db`, while `app` could reach `db:6379`.

![Evidence 4 — Segmented networks](<Evidence/4. Create two segmented networks.png>)

*Figure 4. Creating the frontend and backend networks.*

![Evidence 4.1 — Container network membership](<Evidence/4.1. DB only on backend-net; app on both; web only on frontend-net.png>)

*Figure 4.1. Database, application, and web network assignments.*

![Evidence 4.2 — Segmentation test](<Evidence/4.2. web - db should FAIL.png>)

*Figure 4.2. `web` is blocked from `db`; `app` can reach the database service.*

### 5. Firewall default-deny policy

Within a disposable container granted `NET_ADMIN`, the `INPUT` chain policy was set to `DROP`. Explicit allow rules were added for TCP port 443 and loopback traffic. The displayed rules confirm the default-deny posture and the two exceptions.

![Evidence 5 — iptables rules](<Evidence/5. Inside a throwaway container with iptables, model default-deny + allow 443.png>)

*Figure 5. Default-deny firewall policy with HTTPS and loopback allow rules.*

### 6. Container hardening and image scanning

A service was run with hardening settings. Docker inspection verified that it runs as non-root user/group `1000:1000` and uses a read-only root filesystem. The Nginx Alpine image was then scanned with Trivy; the captured report summary records zero detected vulnerabilities.

![Evidence 6 — Hardened service](<Evidence/6. A hardened run of a service.png>)

*Figure 6. Hardened service container run.*

![Evidence 6.1 — Hardening verification](<Evidence/6.1. Verify hardening settings.png>)

*Figure 6.1. Docker inspection confirms non-root execution and `ReadOnly=true`.*

![Evidence 6.2 — Trivy scan command](<Evidence/6.2. Scan an image for known vulnerabilities.png>)

*Figure 6.2. Starting the Trivy vulnerability scan.*

![Evidence 6.3 — Trivy report](<Evidence/6.3. Scan an image for known vulnerabilities.png>)

*Figure 6.3. Trivy report summary: `nginx:alpine` with 0 detected vulnerabilities.*

### 7. Final verification

The RoleBinding was retrieved in YAML and confirms binding of service account `dev` in namespace `app` to role `dev-role`. Docker inspection also returned `["ALL"]` for the container capability-drop configuration.

![Evidence 7 — Verification](<Evidence/7. Verification Command.png>)

*Figure 7. RoleBinding and container capability-drop verification.*

### 8. Cleanup and teardown

All containers and lab networks were removed, and the Kind cluster `ccse-lab4` was deleted.

![Evidence 8 — Cleanup](<Evidence/8. Cleanup & Teardown.png>)

*Figure 8. Container/network removal and Kind cluster deletion.*

## Conclusion

The lab demonstrated layered cloud-native security controls: credential-based authentication, TOTP validation, least-privilege Kubernetes RBAC, segmented container networks, a default-deny firewall, and hardened, scanned containers. The evidence also confirms resource teardown. The unauthenticated HTTP `200` result is the one item that warrants a configuration check if anonymous access was meant to be denied.
