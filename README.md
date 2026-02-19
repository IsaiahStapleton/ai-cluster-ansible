# Ansible Roles Documentation

This document provides documentation for this ansible collection. Each role is responsible for different aspects of setting up and managing an OpenShift cluster for AI/ML workloads.

## Overview

The project contains several roles that:
1. Build and configure OpenShift clusters
2. Install necessary operators for AI/ML workloads
3. Deploy AI models and supporting infrastructure
4. Set up monitoring and observability
5. Validate AI cluster health/readiness
6. Clean up and uninstall configurations

### Prerequisites

If running locally, you will need to have the `kubernetes` and `ansible` packages installed. You can do this by running the following command from the root directory:

```
pip install -r requirements.txt
```

### Run playbooks

From the root directory, run the following command to run a playbook:

```
ansible-playbook playbooks/[playbook].yaml
```

For example, to run the `configure_managed_cluster` playbook, execute the following command:

```
ansible-playbook playbooks/configure_managed_cluster.yaml
```

## Role Descriptions

### 1. `build_and_configure`

**Purpose**: Creates a new OpenShift cluster and configures it with all necessary operators and features for AI/ML workloads. NOTE: This is used with the [Open Sovereign AI Cloud Solution (OSAC)](https://github.com/osac-project) 


### 2. `configure_managed_cluster`

**Purpose**: Configures an existing managed OpenShift cluster with AI/ML capabilities. "Managed" refers to a cluster connected to an existing observability cluster, so there is no Grafana setup.

### 3. `configure_unmanaged_cluster`

**Purpose**: Configures an existing unmanaged OpenShift cluster with the full AI/ML stack including additional observability tools. This role does everything that `configure_managed_cluster` does, plus:
- Sets up Grafana for metrics visualization
- Configures Grafana with a Prometheus datasource connected to Thanos Querier


### 4. `validate_managed_cluster`

**Purpose**: Validates the health and readiness of a managed AI cluster after configuration.

**What it checks**:
- **Operator Health**: Verifies RHOAI, NFD, NVIDIA GPU, OpenShift Serverless, and Service Mesh operators are all available
- **Core Configurations**: Validates GPU ClusterPolicy, NFD Instance, and DataScienceCluster are in a ready state
- **DataScienceCluster Components**: Confirms all DSC components are ready: Dashboard, Workbenches, Data Science Pipelines, CodeFlare, KServe, Ray, TrustyAI, ModelMesh Serving, Kueue, and ModelRegistry
- **GPU Validation**: Checks GPU node labels (NFD and NVIDIA), NFD worker pods, and NVIDIA driver pods are running
- **Networking**: Validates the Service Mesh (Istio) control plane is healthy
- **Monitoring**: Confirms DCGM Exporter pods are running for GPU metrics

At the end of the run, a summary of all validation results is printed.

### 5. `deploy_workloads`

**Purpose**: Deploys AI/ML workloads and supporting infrastructure.

**What it deploys**:
- **Demo Namespace**: Creates a dedicated namespace for demo workloads
- **MinIO S3 Storage**: Deploys MinIO storage, creates S3 buckets, and sets up data science connections for RHOAI
- **Granite 3.1 2B Instruct Model**: Quantized (w4a16), deployed as a KServe InferenceService using the vLLM runtime
- **vLLM ServingRuntime**: Custom serving runtime for LLM inference
- **Model Upload Job**: Uploads the Granite model to MinIO S3 bucket
- **LLM Load Test Exporter**: Custom metrics exporter with ServiceMonitor for Prometheus integration

### 6. `uninstall_managed_cluster_config`

**Purpose**: Removes everything installed and configured in the cluster from the `configure_managed_cluster` role.

### 7. `uninstall_unmanaged_cluster_config`

**Purpose**: Removes everything installed and configured in the cluster from the `configure_unmanaged_cluster` role.

### 8. `uninstall_workloads`

**Purpose**: Removes everything deployed from the `deploy_workloads` role.

## Installed Operators

The following operators are installed by the configuration roles:

- **Red Hat OpenShift AI (RHOAI)**: Enables model serving (KServe, ModelMesh Serving), Jupyter notebooks (Workbenches), pipelines, model registry, distributed training (Ray, CodeFlare), workload scheduling (Kueue), and AI explainability (TrustyAI)
- **NVIDIA GPU Operator**: Enables GPU acceleration, node labeling, and GPU monitoring
- **Node Feature Discovery (NFD)**: Hardware feature detection and node labeling
- **OpenShift Serverless**: Used for serverless model deployments
- **Service Mesh**: Istio service mesh capabilities
- **Authorino**: Model endpoint security and authorization
- **OpenTelemetry**: Distributed tracing and observability
- **Autopilot**: Health checks for the AI cluster

## Variable

All roles use the variable `kustomize_dir` which points to the `files` directory containing all Kubernetes manifests.
