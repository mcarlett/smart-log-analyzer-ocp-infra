# Smart Log Analyzer OCP - Infrastructure

Tekton pipeline for deploying the observability and messaging infrastructure on OpenShift. This pipeline provisions MinIO, LokiStack, Tempo, Kafka, and an OpenTelemetry Collector.

## Project Structure

```
smart-log-analyzer-ocp-infra/
├── resources/                          # Raw K8s/OpenShift manifests
│   ├── minio/
│   │   ├── pvc.yaml                    # 2Gi PersistentVolumeClaim
│   │   ├── deployment.yaml             # MinIO server
│   │   ├── service.yaml                # API (9000) + Console (9001) services
│   │   └── route.yaml                  # OpenShift Route (TLS edge)
│   ├── lokistack/
│   │   └── lokistack.yaml              # LokiStack CR (1x.demo, S3 backend)
│   ├── tempo/
│   │   ├── tempo.yaml                  # TempoMonolithic CR (multi-tenant)
│   │   └── tempo-roles.yaml            # ClusterRoles/Bindings for traces
│   ├── kafka/
│   │   ├── kafka-nodepool.yaml         # KafkaNodePool (KRaft, 1Gi persistent)
│   │   └── kafka-cluster.yaml          # Kafka cluster (plain + TLS + route)
│   └── otel-collector/
│       ├── serviceaccount.yaml         # ServiceAccount for bearer token auth
│       └── collector.yaml              # OpenTelemetryCollector CR
│
├── tasks/                              # Tekton Task definitions
│   ├── 01-deploy-minio.yaml
│   ├── 02-create-minio-buckets.yaml
│   ├── 03-create-secrets.yaml
│   ├── 04-deploy-lokistack.yaml
│   ├── 05-deploy-tempo.yaml
│   ├── 06-deploy-kafka.yaml
│   └── 07-deploy-otel-collector.yaml
│
├── pipeline/
│   └── infra-pipeline.yaml             # Pipeline orchestrating all tasks
│
└── pipelinerun/
    └── infra-pipeline-run.yaml          # Example PipelineRun
```

## Components

| Component | Description |
|---|---|
| **MinIO** | S3-compatible object storage backend for LokiStack and Tempo |
| **LokiStack** | Log aggregation (Grafana Loki operator, `1x.demo` size) |
| **TempoMonolithic** | Distributed tracing storage (Grafana Tempo operator) |
| **Kafka** | Event streaming via Strimzi (KRaft mode, single node) |
| **OTel Collector** | Receives OTLP telemetry and exports to Loki, Tempo, and Kafka |

## Pipeline Execution Order

```
git-clone
    │
deploy-minio
    │
create-minio-buckets
    │
create-secrets
    │
    ├──────────────┬──────────────┐
    │              │              │
deploy-lokistack  deploy-tempo  deploy-kafka    (parallel)
    │              │              │
    └──────────────┴──────────────┘
                   │
         deploy-otel-collector
```

LokiStack, Tempo, and Kafka are deployed in parallel after secrets are created. The OTel Collector runs last since it exports to all three.

## Prerequisites

- OpenShift 4 cluster
- Tekton CLI (`tkn`) (optional, for monitoring runs)

## Required Operators

The following operators must be installed from the OperatorHub before running the pipeline. All operators should show `Succeeded` phase.

| Operator | Required by |
|---|---|
| **Red Hat OpenShift Pipelines** | Pipeline execution, `git-clone` ClusterTask |
| **Loki Operator** | LokiStack (log aggregation) |
| **Tempo Operator** | TempoMonolithic (distributed tracing) |
| **Streams for Apache Kafka** | Kafka cluster |
| **Red Hat build of OpenTelemetry** | OpenTelemetry Collector |

## Usage

### Deploy the pipeline

```bash
# Create the target namespace
oc new-project camel-otel-infra

# Apply the scoped ClusterRole and ClusterRoleBinding for the pipeline ServiceAccount
# If using a custom namespace, edit resources/rbac/pipeline-clusterrolebinding.yaml first
oc apply -f resources/rbac/

# Apply tasks and pipeline
oc apply -f tasks/
oc apply -f pipeline/

# Start a pipeline run
oc create -f pipelinerun/infra-pipeline-run.yaml
```

### Monitor the run

```bash
# List pipeline runs
tkn pipelinerun list

# Follow logs of the latest run
tkn pipelinerun logs -f -L
```

### Custom namespace

Override the default namespace by editing the PipelineRun or passing parameters:

```bash
tkn pipeline start deploy-infra \
  -p repo-url=https://github.com/mcarlett/smart-log-analyzer-ocp-infra.git \
  -p namespace=my-custom-namespace \
  -w name=shared-workspace,volumeClaimTemplateFile=pipelinerun/infra-pipeline-run.yaml
```

### Alternative: Deploy via Job

A Kubernetes Job can automate the full setup by cloning the repo, applying the Tekton tasks and pipeline, and triggering a PipelineRun. This can be run from anywhere without cloning the repo locally:

```bash
REPO_RAW="https://raw.githubusercontent.com/mcarlett/smart-log-analyzer-ocp-infra/main"

# Create the target namespace
oc new-project camel-otel-infra

# Apply the scoped ClusterRole and ClusterRoleBinding for the pipeline and Job ServiceAccounts
oc apply -f "${REPO_RAW}/resources/rbac/pipeline-clusterrole.yaml"
oc apply -f "${REPO_RAW}/resources/rbac/pipeline-clusterrolebinding.yaml"

# Start the Job (creates the ServiceAccount, clones the repo, applies tasks/pipeline, triggers a PipelineRun, and waits for completion)
oc apply -f "${REPO_RAW}/job/deploy-infra-job.yaml"

# Bind the ClusterRole to the Job's ServiceAccount
oc adm policy add-cluster-role-to-user deploy-infra-pipeline system:serviceaccount:camel-otel-infra:deploy-infra

# Wait for the Job pod to start, then follow logs
oc wait --for=condition=Ready pod -l job-name=deploy-infra --timeout=60s
oc logs -f job/deploy-infra
```

The Job is automatically cleaned up 5 minutes after completion (`ttlSecondsAfterFinished: 300`).

## Data Flow

```
Applications
    │ (OTLP gRPC/HTTP)
    ▼
OTel Collector
    │
    ├──▶ LokiStack    (logs via OTLP/HTTP, mTLS + bearer token)
    ├──▶ Tempo         (traces via OTLP/gRPC, mTLS + bearer token)
    └──▶ Kafka         (logs + traces as OTLP JSON)
```

## Configuration

Default values used by the pipeline:

| Parameter | Default |
|---|---|
| `namespace` | `camel-otel-infra` |
| `repo-revision` | `main` |
| `minio-user` | `minioadmin` |
| `minio-password` | `minioadmin` |
| MinIO storage | 2Gi PVC |
| LokiStack size | `1x.demo` |
| Tempo trace storage | 1Gi |
| Kafka replicas | 1 (1Gi persistent) |
| Log/trace retention | 1 day / 24 hours |

## Cleanup

To remove all deployed resources:

```bash
# Delete the namespace (removes all namespaced resources)
oc delete project camel-otel-infra

# Delete cluster-scoped resources
oc delete clusterrole tempo-traces-reader tempo-traces-write deploy-infra-pipeline
oc delete clusterrolebinding tempo-traces-reader tempo-traces-from-otel deploy-infra-pipeline
```
