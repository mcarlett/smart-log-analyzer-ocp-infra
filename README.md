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
│   │   ├── kafka-nodepool.yaml         # KafkaNodePool (KRaft, ephemeral)
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
- Tekton Pipelines (OpenShift Pipelines operator) installed
- The following operators installed on the cluster:
  - **Loki Operator** (for LokiStack)
  - **Tempo Operator** (for TempoMonolithic)
  - **AMQ Streams / Strimzi** (for Kafka)
  - **OpenTelemetry Operator** (for OTel Collector)
- `git-clone` ClusterTask available
- Tekton CLI (`tkn`) (optional, for monitoring runs)

## Usage

### Deploy the pipeline

```bash
# Create the target namespace
oc new-project camel-otel-infra

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
| Kafka replicas | 1 (ephemeral) |
| Log/trace retention | 1 day / 24 hours |
