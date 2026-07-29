# Lab 0: Environment Setup Report

## Course Information

- Course: IKB42603 Cloud Computing Security Essentials
- Lab: Lab 0 - Environment Setup
- Student Name: Student Name
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

### Step 1: Verify Docker Installation

Purpose:
Docker is required because LocalStack and the kind Kubernetes cluster both run inside containers. It also provides the runtime needed for future lab activities.

Command used:
```bash
docker --version
```

Explanation:
This command confirms that Docker is installed and available in the terminal.

Evidence:

![Docker verification](1.docker.png)

### Step 2: Test Docker with a Sample Container

Purpose:
This verifies that Docker can pull and run containers successfully, not just exist on the system.

Command used:
```bash
docker run --rm hello-world
```

Explanation:
This command downloads and runs a lightweight test container to confirm that Docker is functioning properly.

### Step 3: Verify AWS CLI Installation

Purpose:
AWS CLI is used to send AWS-style commands to LocalStack during the labs without connecting to a real cloud environment.

Command used:
```bash
aws --version
```

Explanation:
This command confirms that the AWS CLI v2 installation is working correctly.

Evidence:

![AWS CLI verification](2.awscli.png)

### Step 4: Verify kind Installation

Purpose:
kind is used to create a local Kubernetes cluster inside Docker.

Command used:
```bash
kind --version
```

Explanation:
This command confirms that kind is installed and available for cluster creation.

Evidence:

![kind verification](3.kind.png)

### Step 5: Verify kubectl Installation

Purpose:
kubectl is used to interact with the Kubernetes cluster after it is created.

Command used:
```bash
kubectl version --client
```

Explanation:
This command confirms that kubectl is installed and ready to communicate with a Kubernetes cluster.

Evidence:

![kubectl verification](3.kubectl.png)

### Step 6: Verify OpenSSL Installation

Purpose:
OpenSSL is used for cryptographic operations such as generating keys and certificates in later labs.

Command used:
```bash
openssl version
```

Explanation:
This command confirms that OpenSSL is installed and available for later security tasks.

Evidence:

![OpenSSL verification](4.helper_tools_openssl.png)

### Step 7: Verify oathtool Installation

Purpose:
oathtool is used to generate one-time password or TOTP values for MFA-related exercises.

Command used:
```bash
oathtool --version
```

Explanation:
This command confirms that the OATH toolkit is installed and usable.

Evidence:

![oathtool verification](4.helper_tools_oath.png)

### Step 8: Start LocalStack Container

Purpose:
LocalStack simulates AWS services locally so that lab exercises can be tested safely without a real AWS account.

Command used:
```bash
docker run -d --name localstack -p 4566:4566 -p 4510-4559:4510-4559 localstack/localstack:3.0
```

Explanation:
This command starts the LocalStack container and exposes the ports needed for AWS-compatible services.

### Step 9: Check LocalStack Health

Purpose:
This confirms that LocalStack is running properly and is ready to receive requests.

Command used:
```bash
curl http://localhost:4566/_localstack/health
```

Explanation:
This command sends a health check to the LocalStack endpoint to verify that the service is up.

### Step 10: Confirm the Running Container

Purpose:
This verifies that the LocalStack container is active and running in Docker.

Command used:
```bash
docker ps
```

Explanation:
This command lists running containers and shows the LocalStack status.

Evidence:

![LocalStack verification](5.localstack.png)

### Step 11: Create the Local Kubernetes Cluster

Purpose:
A local Kubernetes cluster is needed for container orchestration and cluster-based exercises.

Command used:
```bash
kind create cluster --name ccse
```

Explanation:
This command creates a new local Kubernetes cluster named ccse using kind.

### Step 12: Check Cluster Information

Purpose:
This verifies that the cluster control plane is reachable.

Command used:
```bash
kubectl cluster-info --context kind-ccse
```

Explanation:
This command checks whether kubectl can successfully communicate with the running cluster.

### Step 13: View Cluster Nodes

Purpose:
This confirms that the Kubernetes nodes are available and ready.

Command used:
```bash
kubectl get nodes
```

Explanation:
This command lists the nodes in the cluster and shows whether they are ready.

Evidence:

![Kubernetes cluster verification](5.1.kubenetes.png)

### Step 14: Configure AWS CLI Dummy Credentials

Purpose:
The AWS CLI must use dummy credentials so that commands can be directed to LocalStack safely.

Command used:
```bash
aws configure set aws_access_key_id test
```

Explanation:
This command stores a sample access key ID for AWS CLI configuration.

### Step 15: Configure AWS CLI Secret Key

Purpose:
This completes the local dummy credential setup for AWS CLI.

Command used:
```bash
aws configure set aws_secret_access_key test
```

Explanation:
This command stores a sample secret access key for the CLI configuration.

### Step 16: Configure AWS CLI Region

Purpose:
The AWS CLI needs a default region value for commands to be executed consistently.

Command used:
```bash
aws configure set region us-east-1
```

Explanation:
This command sets the default AWS region used by the CLI.

### Step 17: Set the LocalStack Endpoint Variable

Purpose:
This tells the AWS CLI to send commands to the LocalStack endpoint instead of real AWS.

Command used:
```bash
EP='--endpoint-url=http://localhost:4566'
```

Explanation:
This environment variable stores the LocalStack endpoint that will be used in subsequent AWS CLI commands.

### Step 18: Verify AWS CLI with LocalStack

Purpose:
This confirms that AWS CLI can successfully communicate with LocalStack.

Command used:
```bash
aws $EP sts get-caller-identity
```

Explanation:
This command tests the LocalStack connection and verifies that AWS CLI is correctly configured.

Evidence:

![AWS CLI and LocalStack endpoint configuration](6.one-time.png)

## Commands Used

The following commands were used during the lab setup:

```bash
docker --version
docker run --rm hello-world
aws --version
kind --version
kubectl version --client
openssl version
oathtool --version
docker run -d --name localstack -p 4566:4566 -p 4510-4559:4510-4559 localstack/localstack:3.0
curl http://localhost:4566/_localstack/health
docker ps
kind create cluster --name ccse
kubectl cluster-info --context kind-ccse
kubectl get nodes
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```

## Screenshots

The screenshots below provide visual evidence for each major verification step:

- Docker verification: [1.docker.png](1.docker.png)
- AWS CLI verification: [2.awscli.png](2.awscli.png)
- kind verification: [3.kind.png](3.kind.png)
- kubectl verification: [3.kubectl.png](3.kubectl.png)
- OpenSSL verification: [4.helper_tools_openssl.png](4.helper_tools_openssl.png)
- oathtool verification: [4.helper_tools_oath.png](4.helper_tools_oath.png)
- LocalStack verification: [5.localstack.png](5.localstack.png)
- Kubernetes verification: [5.1.kubenetes.png](5.1.kubenetes.png)
- AWS CLI and LocalStack endpoint evidence: [6.one-time.png](6.one-time.png)

## Challenges Encountered

No major blocker was encountered during the documented setup; however, the following issues are commonly encountered when repeating this lab:

- Docker must be running before starting LocalStack or creating the kind cluster.
- Port 4566 may already be in use by another LocalStack instance.
- New terminal sessions may be required after installing tools so that updated PATH settings are loaded.
- kind cluster creation may fail if Docker does not have enough memory or resources available.

## Lessons Learned

- Local lab environments should be verified step by step so that failures can be isolated quickly.
- Using LocalStack and dummy AWS credentials prevents accidental interaction with real cloud resources.
- Container-based tools such as Docker and kind depend heavily on the host system’s available resources.
- Proper documentation and screenshot evidence are important for reproducibility and academic reporting.

## References

- Lab 0 Cheatsheet: [IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf](IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf)
- Example report template: [Lab0-Environment-Setup-EXAMPLE.md](Lab0-Environment-Setup-EXAMPLE.md)
- Docker documentation: https://docs.docker.com/
- AWS CLI documentation: https://docs.aws.amazon.com/cli/
- kind documentation: https://kind.sigs.k8s.io/
- kubectl documentation: https://kubernetes.io/docs/reference/kubectl/
- LocalStack documentation: https://docs.localstack.cloud/

## Conclusion

The lab environment was successfully prepared for future cloud security and Kubernetes exercises. The required tools were installed, verified, and configured to work locally with LocalStack and a kind-managed Kubernetes cluster.
