# Ansible Roles Documentation

This document provides documentation for this ansible collection. Each role is responsible for different aspects of setting up and managing an OpenShift cluster for AI/ML workloads.

## Overview

The project contains several roles that work together to:
1. Deploy and configure OpenShift clusters  
2. Install necessary operators for AI/ML workloads
3. Deploy AI models and supporting infrastructure
4. Set up monitoring and observability
5. Clean up and uninstall configurations

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

**Purpose**: Creates a new OpenShift cluster and configures it with all necessary operators and features for AI/ML workloads.

**What it does**:
- Creates an OpenShift 4.17 cluster using the `cloudkit.templates.ocp_4_17_small` role
- Extracts admin kubeconfig from the cluster
- Installs all required operators (see operators section below)
- Configures NVIDIA GPU support
- Sets up Red Hat OpenShift AI (RHOAI)
- Configures Node Feature Discovery (NFD)
- Sets up Prometheus monitoring
- Configures OpenTelemetry for observability

**Key Configuration Files Used**:
- `files/feature/operators/` - Operator installations
- `files/feature/nvidia/` - NVIDIA GPU configuration
- `files/feature/rhoai/` - RHOAI configuration
- `files/feature/nfd/` - Node Feature Discovery
- `files/feature/prometheus/` - Prometheus monitoring
- `files/feature/opentelemetry/` - OpenTelemetry setup

### 2. `configure_managed_cluster`

**Purpose**: Configures an existing managed OpenShift cluster with AI/ML capabilities. "Managed" refers to a cluster that is connected to an existing observability cluster, so there is no Grafana setup. 

**What it does**:
- Installs all required operators
- Configures NVIDIA GPU support with cluster policy
- Sets up RHOAI with DataScienceCluster
- Configures NFD for hardware feature detection
- Enables Prometheus user workload monitoring
- Sets up OpenTelemetry collector and tracing

**Difference from `build_and_configure`**: Does not create a new cluster, works with existing managed clusters.

### 3. `configure_unmanaged_cluster`

**Purpose**: Configures an existing unmanaged OpenShift cluster with full AI/ML stack including additional observability tools.

**What it does**:
- Everything that `configure_managed_cluster` does, plus:
- Sets up Grafana for visualization
- Configures Grafana with Prometheus datasource
- Creates Grafana-Prometheus service account and token
- Automatically configures Grafana datasource to connect to Thanos Querier

**Additional Features**:
- Full observability stack with Grafana dashboards


### 4. `deploy_workloads`

**Purpose**: Deploys AI/ML workloads and supporting infrastructure.

**What it deploys**:

#### Infrastructure:
- **Demo Namespace**: Creates a dedicated namespace for demo workloads
- **MinIO S3 Storage**: 
  - Deploys MinIO server with 25GB storage
  - Creates S3 buckets for model storage
  - Sets up data science connections for RHOAI integration
  - Configures service accounts and RBAC

#### AI Models:
- **Granite 3.1 2B Instruct Model**:
  - Quantized version (w4a16) for efficient inference
  - Deployed as KServe InferenceService
  - Uses vLLM runtime for high-performance serving
  - Configured with GPU resources (1 GPU, 4-8GB memory)
  - Includes proper tolerations for GPU scheduling

#### Model Management:
- **Model Upload Job**: Uploads the Granite model to MinIO storage
- **vLLM ServingRuntime**: Custom serving runtime for LLM inference

#### Monitoring:
- **LLM Load Test Exporter**: 
  - Custom metrics exporter for LLM performance monitoring
  - Includes ServiceMonitor for Prometheus integration
  - Provides load testing capabilities for deployed models

### 5. `uninstall_managed_cluster_config`

**Purpose**: Removes AI/ML configurations from a managed cluster.

**What it removes**:
- RHOAI DataScienceCluster and related CRDs
- NFD instances
- All installed operators
- ServiceMesh and Authorino ClusterServiceVersions
- Kueue CRDs (multikueue resources)
- Prometheus user workload monitoring configuration


### 6. `uninstall_unmanaged_cluster_config`

**Purpose**: Removes AI/ML configurations from an unmanaged cluster, including observability tools.

**What it removes**:
- Everything that `uninstall_managed_cluster_config` removes, plus:
- Grafana setup and configuration
- Grafana observability configurations
- Additional unmanaged cluster specific resources

### 7. `uninstall_workloads`

**Purpose**: Removes all deployed AI/ML workloads and supporting infrastructure.

**Cleanup Process**:
1. **Model Cleanup**:
   - Removes finalizers from InferenceServices to allow deletion
   - Deletes all InferenceServices (AI models)
   - Removes all ServingRuntimes

2. **Infrastructure Cleanup**:
   - Removes MinIO S3 storage deployment
   - Removes LLM load test exporter
   - Cleans up all MinIO jobs and pods using shell commands



## Installed Operators

The following operators are installed by the configuration roles:

- **Red Hat OpenShift AI (RHOAI)** (`rhods-operator`):
  - Channel: `stable`
  - Source: `redhat-operators`
  - Enables: Jupyter notebooks, model serving, pipelines, distributed training

- **NVIDIA GPU Operator** (`gpu-operator-certified`):
  - Source: `certified-operators`
  - Enables: GPU acceleration, device plugins, monitoring

- **Node Feature Discovery (NFD)** (`nfd`):
  - Channel: `stable`
  - Source: `redhat-operators`
  - Enables: Hardware feature detection and labeling

- **OpenShift Serverless** (`serverless-operator`):
  - Channel: `stable`
  - Source: `redhat-operators`
  - Enables: Knative Serving for model inference

- **Service Mesh** (`servicemesh-operator`):
  - Provides: Istio service mesh capabilities

- **Authorino** (`authorino-operator`):
  - Provides: API security and authorization

- **OpenTelemetry** (`opentelemetry-product`):
  - Channel: `stable`
  - Source: `redhat-operators`
  - Enables: Distributed tracing and observability

- **Autopilot**:
  - Provides: Provides health checks for your AI cluster.

## Key Configurations

### RHOAI DataScienceCluster Components:
- **Enabled**: CodeFlare, KServe, TrustyAI, Ray, Kueue, Workbenches, Dashboard, ModelMesh Serving, Data Science Pipelines
- **KServe Configuration**: Uses OpenShift default ingress certificates

### NVIDIA GPU Configuration:
- **Driver**: Enabled with auto-upgrade policy
- **DCGM Exporter**: Enabled with ServiceMonitor for Prometheus
- **Device Plugin**: Enabled with MPS support
- **GPU Feature Discovery**: Enabled
- **MIG Manager**: Enabled for multi-instance GPU support

### Monitoring Stack:
- **Prometheus**: User workload monitoring enabled
- **OpenTelemetry**: Collector configured for traces and metrics
- **Grafana** (unmanaged clusters): Integrated with Prometheus/Thanos
- **Custom Metrics**: LLM load test exporter for model performance

### Storage Configuration:
- **MinIO**: S3-compatible storage for model artifacts
- **Data Science Connections**: Automated setup for RHOAI integration
- **Persistent Storage**: 25GB PVC for MinIO data

## Variable

All roles use the variable `kustomize_dir` which points to the `files` directory containing all Kubernetes manifests. 
