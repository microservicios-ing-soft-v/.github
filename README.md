# Azure Microservices CI/CD Architecture

## Overview

This project describes a complete CI/CD architecture deployed on **Azure**, based on a **microservices architecture**.  
It uses separate GitHub repositories, GitHub Actions for CI/CD pipelines, Terraform for infrastructure as code, Ansible for configuration management, and Azure services like Virtual Machines, Docker, and Docker Compose.

All microservices follow well-established architectural patterns:  
- **API Gateway Pattern** for centralized request routing and access control.  
- **Health Check Pattern** for active health monitoring and auto-recovery of services.

---

## Architecture Components

### Repositories

This solution uses a **multi-repository model**:

- **Infrastructure Repository**:
  - Contains Terraform scripts, Ansible configurations, and Docker Compose files.
  - Manages all infrastructure provisioning through GitHub Actions.
  - Connects to a previously configured **Azure Storage Account and Blob Container** to store the Terraform `tfstate` file, ensuring state consistency across executions.

- **Microservices Repositories**:
  - One repository per microservice:
    - `frontend`
    - `api-gateway`
    - `auth-api`
    - `users-api`
    - `todos-api`
    - `log-message-processor`
  - Each repository contains its **own GitHub Actions CI/CD pipeline**, which is responsible for:
    - Building Docker images using **multi-stage Dockerfiles optimized with Alpine**.
    - Pushing images to **Azure Container Registry (ACR)**.
    - Deploying the service via **Docker Compose** on the Azure Virtual Machine.

---

## Azure Authentication and Permissions

A **Service Principal** was created using an **App Registration** in Azure Active Directory. This identity is used by GitHub Actions pipelines to authenticate securely with Azure.

- The Service Principal is granted:
  - Contributor role on the Azure subscription/resource group (for infrastructure provisioning).
  - **ACR Push** role on the Azure Container Registry (to upload Docker images).

Secrets related to this Service Principal (client ID, tenant ID, secret) are securely stored as GitHub Actions secrets and injected at runtime into pipelines.

---

## Infrastructure Deployment (Terraform)

The infrastructure repository provisions the following using **Terraform**:

- **Azure Virtual Machine (VM)**:
  - Hosts Docker and Docker Compose.
  - Acts as the runtime environment for all microservices.
  - After provisioning, the **infrastructure pipeline executes an Ansible playbook** to automatically install and configure required dependencies on the VM, including **Docker**, **Docker Compose**, and other system packages essential for container orchestration and service execution.

- **Azure Container Registry (ACR)**:
  - Stores Docker images pushed by CI pipelines.

- **Azure Key Vault**:
  - Acts as an **external configuration store**.
  - Stores the **public IP address of the VM**, which is dynamically retrieved by microservices pipelines during deployment.

- **Azure Storage Account with Blob Container**:
  - Used to store the **Terraform state file (`tfstate`)**, ensuring consistent state tracking across infrastructure changes.

- **Network Security Group (NSG) Rules**:
  - Configured to allow:
    - HTTP traffic to the **frontend** (port 8080).
    - Access to the **Zipkin** interface (port 9411).
  - Blocks unnecessary ports to secure the VM.

- The infrastructure pipeline is triggered by pushes to the `main` branch of the infrastructure repository and executes the following steps:
  1. Authenticate with Azure using the Service Principal.
  2. Connect to the remote `tfstate` backend.
  3. Apply the Terraform configuration.
  4. Store the public IP of the provisioned VM in **Azure Key Vault**.

---

## CI/CD Workflow (GitHub Actions)

### Infrastructure Pipeline

- **Trigger**: Push to `main` branch.
- **Steps**:
  1. Authenticate to Azure using the Service Principal.
  2. Apply Terraform scripts using remote state backend.
  3. Save the public IP of the Azure VM into Azure Key Vault.

### Microservices Pipelines

Each microservice has a GitHub Actions pipeline with the following workflow:

- **Trigger**: Push to `main` branch.
- **Steps**:
  1. Build the Docker image using a **multi-stage Dockerfile** with **Alpine-based minimal runtime**.
  2. Authenticate with Azure using the Service Principal.
  3. Push the Docker image to **Azure Container Registry (ACR)**.
  4. Retrieve the public IP of the Azure VM from **Azure Key Vault**.
  5. SSH into the VM and restart the corresponding container using **Docker Compose**, pulling the latest image from ACR.

---

## Microservices and Communication

| Service                | Technology                  | Purpose                                                                                       |
|-------------------------|------------------------------|-----------------------------------------------------------------------------------------------|
| Frontend                | Vue.js + Node.js + NPM       | Serves web application and acts as reverse proxy to `Zipkin` and `api-gateway`.               |
| API Gateway             | Node.js + Express            | Routes client requests to `auth-api` or `todos-api`, with circuit breaker and retry logic.    |
| Auth API                | Golang                       | Authenticates users via `users-api` and generates JWT tokens.                                 |
| Users API               | Java + Spring Boot           | Manages user data for authentication and validation.                                          |
| Todos API               | Node.js + Express            | Manages user tasks stored in `redis`.                                                         |
| Redis                   | Redis container              | Stores user tasks. **Deployed as an internal container**, not as a managed Azure service.     |
| Log Message Processor   | Python                       | Reads and logs events from `redis`.                                                           |
| Zipkin                  | Distributed Tracing Tool     | Receives traces from all services (except `redis` and `api-gateway`).                         |

---

## Architectural Patterns

### API Gateway Pattern

- The **API Gateway** acts as a single entry point for the frontend.
- Routes requests to backend services (`auth-api`, `todos-api`) based on URL paths.
- Implements a **circuit breaker** pattern to handle failed requests: after a certain number of failed requests, the gateway prevents further requests and retries them after a delay.
- Benefits:
  - Simplifies frontend client.
  - Centralizes authentication, logging, and routing.
  - Enables easier scalability and security.
  - Provides a safeguard against backend failures.

### Health Check Pattern

- Each microservice implements a **health check endpoint** (e.g., `/health`).
- Azure Virtual Machine uses these endpoints to automatically:
  - Monitor service health.
  - Restart unhealthy containers.
  - Route traffic only to healthy instances.

### External Configuration Store Pattern

- Application configuration values, such as the Azure VM IP address, are stored securely in **Azure Key Vault**.
- The infrastructure pipeline saves VM details into the Key Vault.
- Code pipelines retrieve these configurations during deployment or runtime.

**Benefits**:
- Decouples configuration from application code.
- Enables dynamic updates without redeploying services.
- Strengthens security by keeping sensitive values out of code repositories.

---

## Infrastructure Diagram

![Architecture Diagram](../diagram.png)

---

## Summary

This architecture offers:

- Full CI/CD automation with GitHub Actions.
- Declarative and repeatable infrastructure management with Terraform.
- Strong separation of concerns using microservices and multi-repo structure.
- Remote state management with Azure Storage for safe and collaborative Terraform usage.
- Secure and dynamic configuration handling with Azure Key Vault.
- Lightweight and production-optimized Docker images using Alpine and multi-stage builds.
- Secure image delivery to ACR with proper Service Principal permissions.
- Automated service updates via remote container restart on a Docker-based VM.
- Monitoring and automatic healing through health checks.
- Circuit breaker logic in the API Gateway for service resilience.
- Scalable and modular deployment using Azure Virtual Machine, Docker, and Docker Compose.
