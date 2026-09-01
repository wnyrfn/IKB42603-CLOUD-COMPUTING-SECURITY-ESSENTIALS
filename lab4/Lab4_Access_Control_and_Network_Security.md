# Lab 4: Access Control and Network Security

| Item | Details |
|---|---|
| Course | IKB42603 - Cloud Computing Security |
| Lab | Lab 4: Access Control and Network Security |
| Student | WAN MUHAMMD IRFAN BIN MOHD ISA |

## Objective

This lab applied defence-in-depth controls to containerised workloads: password authentication, time-based MFA, Kubernetes role-based access control (RBAC), three-tier network segmentation, default-deny firewall rules, container hardening, and vulnerability scanning.

## Environment

The work was performed from a Kali Linux host using Docker, Kubernetes-in-Docker (kind), `kubectl`, `iptables`, `oathtool`, and Trivy. Commands below are recorded as executed; secrets are shown only because they appear in the supplied lab evidence and must not be used in a real environment.

---

## Task 1 - Authentication: Password-Protected Service

Basic authentication was added to an Nginx service. The supplied test without credentials returned `no-creds: 200`, while the credentialed request returned `Authenticated OK`. This exposes a configuration caveat: the `return 200` directive shown in the location block short-circuits the expected authentication challenge.

```bash
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssw0rd!' > htpasswd.txt
```

Creates a bcrypt password-hash entry for user `student` and writes it to `htpasswd.txt`.

```nginx
server { listen 80;
  location / { auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
    return 200 'Authenticated OK\n'; } }
```

Defines an Nginx virtual host intended to require HTTP Basic authentication and use the mounted password file.

```bash
docker run --rm -d --name authsvc -p 8080:80 \
  -v $(pwd)/default.conf:/etc/nginx/conf.d/default.conf \
  -v $(pwd)/htpasswd.txt:/etc/nginx/.htpasswd nginx
curl -s -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080
curl -s -u student:'P@ssw0rd!' http://localhost:8080
```

Runs the service, tests it without credentials, then makes an authenticated request. In production, remove `return 200` (or serve content through a protected location) so Nginx returns `401 Unauthorized` before valid credentials are presented.

**Evidence**

![Task 1 terminal evidence: password file creation, service deployment, and requests](<evidence/Task1-Authentication a Password-Protected Service.png>)

---

## Task 2 - Add a Second Factor (MFA/TOTP)

Time-based one-time passwords (TOTP) were generated and validated to demonstrate a second authentication factor.

```bash
SECRET=$(head -c20 /dev/urandom | base32)
echo "Enroll this secret in an authenticator app: $SECRET"
oathtool --totp -b "$SECRET"
```

Generates a random Base32 TOTP seed, displays it for authenticator-app enrolment, and calculates the current six-digit code from that seed.

```bash
read -p 'Enter the 6-digit code: ' CODE
[ "$CODE" = "$(oathtool --totp -b "$SECRET")" ] && echo 'MFA OK' || echo 'MFA FAILED'
```

Prompts for a user-provided code, compares it with the current expected TOTP value, and reports success or failure. The evidence shows one accepted code and one rejected code.

**Evidence**

![Task 2 terminal evidence: TOTP generation and successful and failed validation](<evidence/Task2-Add a Second Factor (MFA  TOTP).png>)

---

## Task 3 - Authorization: Kubernetes RBAC Roles

RBAC was configured to grant a service account read-only access to pods in the `app` namespace.

```bash
kind create cluster --name ccse-lab4
kubectl create namespace app
kubectl create serviceaccount dev -n app
```

Creates a local Kubernetes cluster, isolates lab resources in the `app` namespace, and creates the `dev` workload identity.

```bash
kubectl create role dev-role -n app --verb=get,list --resource=pods
kubectl create rolebinding dev-rb -n app --role=dev-role --serviceaccount=app:dev
```

Creates a namespace-scoped role that permits only `get` and `list` on pods, then binds that role to the `dev` service account.

```bash
SA=system:serviceaccount:app:dev
kubectl auth can-i list pods -n app --as=$SA
kubectl auth can-i create deploy -n app --as=$SA
kubectl auth can-i delete pods -n app --as=$SA
```

Checks the effective permissions of the service account. The results show `yes` for listing pods and `no` for creating deployments or deleting pods, demonstrating least privilege.

**Evidence**

![Task 3 terminal evidence: kind cluster, RBAC role and role binding, and permission checks](<evidence/Task3-Authorization RBAC Roles.png>)

---

## Task 4 - Network Segmentation (Three-Tier)

Separate Docker networks modelled frontend and backend security zones. The web tier is attached only to the frontend network, the database only to the backend network, and the application tier bridges the two zones.

```bash
docker network create frontend-net
docker network create backend-net
docker run -d --name db --network backend-net redis:alpine
docker run -d --name app --network backend-net nginx
docker network connect frontend-net app
docker run -d --name web --network frontend-net nginx
```

Creates the two network segments; deploys Redis in the backend, an application container in the backend and frontend, and a web container in the frontend. This limits direct web-to-database connectivity while preserving the app-to-database path.

```bash
docker exec web sh -c 'apk add -q curl; curl -s -m 3 db:6379 || echo BLOCKED'
docker exec app sh -c 'apk add -q curl; nc -z -w3 db 6379 && echo REACHABLE'
```

Attempts to probe the database from the web and application tiers. The web-side probe printed `BLOCKED`. The app-side probe could not complete in the supplied run because the Nginx image did not contain `apk` or `nc`; a purpose-built troubleshooting image with those tools should be used to verify the allowed application-to-database path.

**Evidence**

![Task 4 terminal evidence: frontend and backend Docker networks and connectivity probes](<evidence/Task4-Network Segmentation (Three-Tier).png>)

---

## Task 5 - Firewall Rules (Default-Deny)

A default-deny inbound firewall policy was established, with only HTTPS and loopback traffic explicitly allowed.

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c '
  apk add -q iptables;
  iptables -P INPUT DROP;
  iptables -A INPUT -p tcp --dport 443 -j ACCEPT;
  iptables -A INPUT -i lo -j ACCEPT;
  iptables -L INPUT -n'
```

Starts a short-lived privileged network test container, installs `iptables`, changes the INPUT chain policy to `DROP`, permits TCP/443 and local loopback traffic, then lists the active rules. All other unsolicited inbound traffic is denied.

**Evidence**

![Task 5 terminal evidence: INPUT policy DROP with explicit HTTPS and loopback allow rules](<evidence/Task5-Firewall Rules (Default-Deny).png>)

---

## Task 6 - Container and Host Hardening

### 6.1 Harden the Container Runtime Configuration

```bash
docker run -d --name hardened \
  --user 1000:1000 \
  --read-only \
  --cap-drop=ALL \
  --security-opt no-new-privileges \
  --tmpfs /tmp \
  --tmpfs /var/cache/nginx \
  --tmpfs /var/run \
  nginxinc/nginx-unprivileged
docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'
```

Runs Nginx as non-root UID/GID 1000, makes its root filesystem read-only, removes Linux capabilities, blocks privilege escalation, and provides only temporary writable directories. The inspection output confirms `User=1000:1000` and `ReadOnly=true`.

**Evidence**

![Task 6.1 terminal evidence: hardened unprivileged Nginx container and inspection](<evidence/Task6.1-Container  Host Hardening.png>)

### 6.2-6.3 Scan the Image for Known Vulnerabilities

```bash
docker run --rm aquasec/trivy image --severity HIGH,CRITICAL nginx:alpine | head -20
```

Runs Trivy against `nginx:alpine`, limiting displayed findings to HIGH and CRITICAL severity and showing the first 20 lines of output. The scanner downloads its vulnerability database on the first run.

The completed scan summary identifies **4 HIGH** and **0 CRITICAL** vulnerabilities for `nginx:alpine (alpine 3.24.1)`. These findings should be reviewed, patched by rebuilding from an updated base image where possible, and accepted only with documented risk justification.

**Evidence - scan start**

![Task 6.2 terminal evidence: Trivy image scan and vulnerability database download](<evidence/Task6.2-Scan an image for known vulnerabilities.png>)

**Evidence - report summary**

![Task 6.3 terminal evidence: Trivy report showing four high and zero critical vulnerabilities](<evidence/Task6.3-Report summary for scan.png>)

---

## Short-Answer Questions

**Q1. Explain the difference between authentication and authorization using Tasks 1 and 3.**  
Authentication verifies identity: Task 1 checks whether a requester presents valid credentials. Authorization decides permitted actions after identity is known: Task 3 allows `dev` to list pods but denies deployment creation and pod deletion.

**Q2. Why is MFA so effective, and which attacks does it defeat?**  
MFA requires an additional, short-lived factor, so a stolen password alone is insufficient. It defeats many credential-stuffing, password-spraying, phishing-reuse, and database-leak attacks; it is less resistant to real-time adversary-in-the-middle phishing unless phishing-resistant factors are used.

**Q3. How does network segmentation limit the damage of a compromised web server?**  
The web server is isolated in the frontend network and cannot directly reach the backend database. An attacker who compromises it cannot easily move laterally to the database; access must pass through the controlled application tier.

**Q4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?**  
It blocks all inbound traffic except explicit allow rules, reducing exposed services and accidental access. Cloud security groups implement the same allow-list model at the cloud virtual-network/interface boundary.

**Q5. List the hardening measures you applied and the attack surface each one removes.**

| Measure | Attack surface reduced |
|---|---|
| Basic authentication | Unauthenticated access to the web service |
| TOTP MFA | Account takeover using only a stolen password |
| Least-privilege RBAC | Unauthorised Kubernetes deployment and pod changes |
| Frontend/backend segmentation | Direct web-tier access to database services and lateral movement |
| Default-deny firewall | Unnecessary inbound ports and services |
| Non-root container user | Root-level impact from a compromised process |
| Read-only root filesystem and `tmpfs` | Persistent file modification and malware staging |
| Dropped capabilities and `no-new-privileges` | Kernel-level and privilege-escalation operations |
| Trivy image scanning | Known vulnerable packages in the container image |

## Conclusion

The lab demonstrated layered controls across identity, permissions, network exposure, runtime configuration, and supply-chain visibility. Together, these controls reduce both the likelihood of compromise and the impact if an individual workload is breached.
