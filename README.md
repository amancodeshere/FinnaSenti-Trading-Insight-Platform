# FinnaSenti Trading Insight Platform

AI system that Analyses news, tweets, and filings to generate sentiment-driven trading signals. 

- __Status__: Ground-up build.
- __Stack__: React and Tailwind (frontend), Node.js with TypeScript (orchestrator and ingestion), Python (NLP), C++ (signals and execution), PostgreSQL (RDS), S3, SQS, ECS Fargate (AWS-first deployment).
- __Interoperability__: Contract-first gRPC with Protocol Buffers for communication between TypeScript, Python, and C++.

---
### Primary Objectives
- Transform unstructured financial text (news, tweets, filings) into normalised sentiment and event features.
- Generate trading signals and execute backtests using NLP-derived features and price data.
- Provide a React and Tailwind dashboard for configuration, analysis, and visualization.
- Maintain strict language boundaries across components using Protobuf/gRPC.
- Begin with a minimal functional prototype and improve iteratively in structured phases.

### Non-Objectives (initially)
- Live integration with broker APIs.
- Sub-millisecond latency for execution.
- Full-scale data coverage. Initial builds will focus on limited datasets and sample tickers.

---

### Services
- __Frontend (React + Tailwind)__
  - Displays dashboards for portfolios, signals, and backtest reports.
- __Backend API (Node.js/TypeScript)__
  - Central gateway for the application. Manages user sessions, backtest jobs, and orchestration of NLP and signal services.
  - Ingestion Service (Node.js/TypeScript)
  - Gathers raw data from APIs or feeds, normalises text and metadata, stores raw files in S3, and maintains metadata in PostgreSQL.
- __NLP Service (Python)__
  - Provides FinBERT or LLM-based sentiment analysis and event extraction through gRPC.
- __Signal Engine (C++)__
  - Computes low-latency factors and trading signals. Includes an order execution simulator.
  - Backtest Coordinator (TypeScript)
  - Manages backtest workflows, retrieves historical data, calls the C++ engine, and aggregates results to S3.
 
- __Data Storage__
  - S3: Stores raw documents, model files, and backtest reports.
  - PostgreSQL (RDS): Stores structured metadata including users, documents, NLP outputs, and signals.
  - OpenSearch: Enables text-based search across the corpus.
- __Messaging__
  - SQS: Used for distributed backtest or ingestion tasks.
  - EventBridge: Handles scheduled jobs such as daily data refresh or model updates.

---

### Cross-Language Communication

All cross-service communication follows a contract-first Protobuf model. This ensures compatibility, performance, and schema evolution across TypeScript, Python, and C++.


nlp.proto
```nlp.proto
syntax = "proto3";
package nlp;

message Document {
  string id = 1;
  string source = 2;
  string text = 3;
  string ticker = 4;
  int64  published_at = 5;
}

message SentimentScore {
  double polarity = 1;
  double confidence = 2;
  string model_version = 3;
}

message EventTag {
  string type = 1;
  double confidence = 2;
}

message AnalyseResponse {
  SentimentScore sentiment = 1;
  repeated EventTag events = 2;
  repeated string entities = 3;
}

service NlpService {
  rpc Analyse (Document) returns (AnalyseResponse);
  rpc AnalyseStream (stream Document) returns (stream AnalyseResponse);
}
```

signal.proto
```signal.proto
syntax = "proto3";
package signal;

message Bar {
  string ticker = 1;
  int64  ts = 2;
  double open = 3;
  double high = 4;
  double low  = 5;
  double close= 6;
  double vol  = 7;
}

message FactorInput {
  repeated Bar bars = 1;
  double sentiment = 2;
  double event_impact = 3;
}

message Signal {
  string ticker = 1;
  double score = 2;
  double strength = 3;
}

message ComputeSignalsRequest { repeated FactorInput inputs = 1; }
message ComputeSignalsResponse { repeated Signal signals = 1; }

service SignalEngine {
  rpc ComputeSignals (ComputeSignalsRequest) returns (ComputeSignalsResponse);
  rpc RunExecutionSim (stream Bar) returns (stream Signal);
}
```

---

### Prototype Setup

__Prerequisites:__
| Requirement                            | Version / Details           | Purpose                                                                     |
| -------------------------------------- | --------------------------- | --------------------------------------------------------------------------- |
| **Node.js**                            | **20+**                     | Runtime for TypeScript backend services (Orchestrator, Ingestion, Backtest) |
| **TypeScript**                         | **5.x or later**            | Strongly-typed backend logic and API contracts                              |
| **Python**                             | **3.11+**                   | NLP service (FinBERT, summarisation, sentiment extraction)                  |
| **C++**                                | **C++20 + CMake 3.25+**     | High-performance signal computation and simulation                          |
| **Docker & Docker Compose**            | Latest                      | Containerised development environment                                       |
| **Protocol Buffers Compiler (protoc)** | **3.21+ with gRPC plugins** | Generate shared API contracts (TypeScript, Python, and C++)                 |
| **PostgreSQL**                         | **15+**                     | Primary database                                      |


__Repo Layout:__
```
FinnaSenti-Trading-Insight-Platform/
  proto/
  orchestrator-ts/
  ingestion-ts/
  nlp-py/
  signal-cpp/
  backtest-ts/
  frontend/
  docker/
  infra/
  scripts/
  docs/
```

__Generating gRPC Stubs__
```npm run proto
npm run proto:gen:ts
python -m grpc_tools.protoc -I=proto \
  --python_out=nlp-py/generated \
  --grpc_python_out=nlp-py/generated proto/nlp.proto
protoc -I=proto --cpp_out=signal-cpp/generated --grpc_out=signal-cpp/generated \
  --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` proto/signal.proto
```

__ENV Variables to be aware of__
| Variable                          | Purpose                               |
| --------------------------------- | ------------------------------------- |
| NLP_HOST / NLP_PORT               | gRPC address for NLP service          |
| SIGNAL_HOST / SIGNAL_PORT         | gRPC address for C++ service          |
| DATABASE_URL                      | PostgreSQL connection string          |
| AWS_REGION                        | AWS region for resources              |
| S3_BUCKET_RAW / S3_BUCKET_REPORTS | Bucket names for raw data and reports |
| JWT_ISSUER / JWT_AUDIENCE         | Authentication configuration          |

__API Endpoints (Prototype)__

__REST__
- `POST /api/documents/Analyse` – calls Python NLP service
- `POST /api/signals/compute` – calls C++ signal engine
- `POST /api/backtests` -> initiates a backtest job
- `GET /api/backtests/:id` -> retrieves backtest status and summary

---

### Prototype Database Schema
```sql
CREATE TABLE documents (
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL,
  ticker TEXT,
  s3_key TEXT NOT NULL,
  published_at BIGINT NOT NULL,
  ingested_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE nlp_results (
  doc_id TEXT REFERENCES documents(id) ON DELETE CASCADE,
  polarity DOUBLE PRECISION,
  confidence DOUBLE PRECISION,
  model_version TEXT,
  entities JSONB,
  events JSONB,
  PRIMARY KEY (doc_id)
);

CREATE TABLE backtests (
  id UUID PRIMARY KEY,
  user_id TEXT,
  status TEXT NOT NULL,
  started_at TIMESTAMP,
  finished_at TIMESTAMP,
  summary JSONB,
  s3_report_key TEXT
);
```

---

### Coding Standards

- TypeScript
  - ESLint and Prettier enforced.
  - Callback-based patterns for gRPC client wrappers.
- Python
  - Style checks with ruff and black.
  - All inference outputs must include model version metadata.
- C++
  - C++20 standard.
  - Use value semantics, preallocate buffers, and ensure deterministic output.
- Protobuf
  - Add new fields only; never reuse field numbers.

---

### Security Practices
- JWT-based authentication verified by API Gateway.
- Minimal IAM privileges for each ECS task.
- Secrets stored only in AWS Secrets Manager.
- S3 private by default, accessed through signed URLs.

---

### Project Setup Commands
```
npm run proto:gen:ts
make proto:py
make proto:cpp
docker compose build
docker compose up
npm run test -w orchestrator-ts
pytest nlp-py/tests
ctest --test-dir signal-cpp/build
```
