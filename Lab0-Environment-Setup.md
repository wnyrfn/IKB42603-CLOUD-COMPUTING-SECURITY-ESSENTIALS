# Lab 0: Environment Setup Report

## Course Information

- Course: IKB42603 Cloud Computing Security Essentials
- Lab: Lab 0 - Environment Setup
- Student Name: WAN MUHAMMAD IRFAN BIN MOHD ISA
- Date: 29 July 2026

## Objective

The objective of this lab was to prepare a local cybersecurity/cloud security lab environment that can support Docker-based services, AWS CLI commands directed to LocalStack, a local Kubernetes cluster created with kind, and the helper tools required for later labs. The setup ensures that the workstation is ready for secure container, cloud, and Kubernetes exercises without using a real AWS account.

## Learning Outcomes

By completing this lab, the student should be able to:

- Install and verify the core tools required for the course labs.
- Understand the purpose of Docker, AWS CLI, kind, kubectl, OpenSSL, and oathtool in a local lab environment.
- Configure AWS CLI to communicate with LocalStack using dummy credentials.
- Create and verify a local Kubernetes cluster using kind.
- Document technical setup steps with evidence and explain the purpose of each command.

## Environment

The environment used for this lab included a local workstation with support for container runtime and command-line tooling. The following components were verified during setup:

- Docker for running containers and LocalStack
- AWS CLI v2 for interacting with LocalStack
- kind and kubectl for creating and managing a local Kubernetes cluster
- OpenSSL and oathtool for later cryptography and MFA-related work
- LocalStack as a local AWS-compatible service simulator

## Step-by-Step Implementation

### 1. Install and Verify Docker

Purpose:
Docker is required because LocalStack and the kind Kubernetes cluster both run inside containers. It also provides the runtime needed for future lab activities.

Commands used:
```bash
docker --version
docker run --rm hello-world
```

Explanation:

| Command | Explanation |
| --- | --- |
| `docker --version` | Confirms that Docker is installed and accessible on the system. |
| `docker run --rm hello-world` | Verifies that Docker can pull and run a container successfully. |

Evidence:

<img width="488" height="222" alt="1 docker" src="https://github.com/user-attachments/assets/d7113aca-8347-43eb-a90a-cf5cf9903dfb" />

### 2. Install and Verify AWS CLI v2

Purpose:
AWS CLI is used to send AWS-style commands to LocalStack during the labs. This allows the student to practice cloud commands without connecting to real AWS services.

Commands used:
```bash
aws --version
```

Explanation:

| Command | Explanation |
| --- | --- |
| `aws --version` | Verifies that AWS CLI v2 is installed correctly and ready to use. |

Evidence:

<img width="485" height="81" alt="2 awscli" src="https://github.com/user-attachments/assets/eaea5227-0b9e-47f9-94d2-ed97165ea464" />

### 3. Install and Verify kind and kubectl

Purpose:
kind creates a local Kubernetes cluster inside Docker, while kubectl is used to interact with that cluster. These tools are essential for Kubernetes-based labs.

Commands used:
```bash
kind --version
kubectl version --client
```

Explanation:

| Command | Explanation |
| --- | --- |
| `kind --version` | Confirms that kind is installed and can create local Kubernetes clusters. |
| `kubectl version --client` | Confirms that kubectl is installed and functioning for client-side Kubernetes operations. |

Evidence:

<img width="482" height="69" alt="3 kind" src="https://github.com/user-attachments/assets/2658cfcf-c5d1-485e-b8db-1323bffbd18e" />


<img width="481" height="93" alt="3 kubectl" src="https://github.com/user-attachments/assets/30484fca-d8f3-4a79-ae14-a703ec53b676" />

### 4. Install and Verify Helper Tools

Purpose:
OpenSSL and oathtool support later labs involving keys, certificates, and MFA/TOTP-related tasks. For Trivy	Container vulnerability scanning, will run through Docker in Lab 4.

Commands used:
```bash
openssl version
oathtool --version
```

Explanation:

| Command | Explanation |
| --- | --- |
| `openssl version` | Confirms that OpenSSL is installed and available for cryptographic tasks. |
| `oathtool --version` | Confirms that the OATH toolkit is installed for TOTP/MFA-related operations. |

Evidence:

<img width="476" height="74" alt="4 helper_tools_openssl" src="https://github.com/user-attachments/assets/d3e08b0b-735c-498b-9f10-39166c545961" />


<img width="487" height="181" alt="4 helper_tools_oath" src="https://github.com/user-attachments/assets/eab1a61a-1fcd-456c-92c6-13504256d0d4" />

### 5. Start and Verify LocalStack

Purpose:
LocalStack simulates AWS services locally so that lab exercises can be tested without a live cloud account. It provides a safe and isolated environment for AWS CLI practice.

Commands used:
```bash
docker run -d --name localstack -p 4566:4566 -p 4510-4559:4510-4559 localstack/localstack:3.0
curl http://localhost:4566/_localstack/health
docker ps
```

Explanation:

| Command | Explanation |
| --- | --- |
| `docker run -d --name localstack -p 4566:4566 -p 4510-4559:4510-4559 localstack/localstack:3.0` | Starts LocalStack in detached mode and exposes the AWS-compatible ports. |
| `curl http://localhost:4566/_localstack/health` | Verifies that the LocalStack health endpoint is responding. |
| `docker ps` | Confirms that the LocalStack container is running and active. |

Evidence:

<img width="477" height="268" alt="5 localstack" src="https://github.com/user-attachments/assets/4d9d052d-d15d-476d-b351-e33946c68878" />

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/dff574a0-cdce-4cec-966f-2c2ffaf3fcb4" />


### 6. Create and Verify the Kubernetes Cluster

Purpose:
A local Kubernetes cluster is needed for container orchestration exercises and to simulate a real cluster environment on the student machine.

Commands used:
```bash
kind create cluster --name ccse
kubectl cluster-info --context kind-ccse
kubectl get nodes
```

Explanation:

| Command | Explanation |
| --- | --- |
| `kind create cluster --name ccse` | Creates the local Kubernetes cluster named `ccse` inside Docker. |
| `kubectl cluster-info --context kind-ccse` | Verifies that the Kubernetes control plane is reachable. |
| `kubectl get nodes` | Confirms that the cluster node is running and ready. |

Evidence:

<img width="480" height="253" alt="5 1 kubenetes" src="https://github.com/user-attachments/assets/dc3af012-4296-48dd-ab85-801fd4117384" />

### 7. Configure AWS CLI to Use LocalStack

Purpose:
The AWS CLI should point to LocalStack instead of real AWS so that all commands remain local and safe.

Commands used:
```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```

Explanation:

| Command | Explanation |
| --- | --- |
| `aws configure set aws_access_key_id test` | Stores a dummy access key for local AWS CLI use. |
| `aws configure set aws_secret_access_key test` | Stores a dummy secret access key for local AWS CLI use. |
| `aws configure set region us-east-1` | Sets the region used by the CLI for local commands. |
| `EP='--endpoint-url=http://localhost:4566'` | Defines the LocalStack endpoint so AWS CLI targets the local service. |
| `aws $EP sts get-caller-identity` | Verifies that AWS CLI can successfully communicate with LocalStack. |

Evidence:

<img width="484" height="280" alt="6 one-time" src="https://github.com/user-attachments/assets/b25044da-50ef-449b-992d-2341af872970" />

## Conclusion

The lab environment was successfully prepared for future cloud security and Kubernetes exercises. The required tools were installed, verified, and configured to work locally with LocalStack and a kind-managed Kubernetes cluster.
