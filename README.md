# Binance Analytics Pipeline

> A self-hosted trading analytics platform for Binance Spot — place orders and understand your real performance through a full AWS data pipeline.

![Python](https://img.shields.io/badge/Python-3.12-blue)
![dbt](https://img.shields.io/badge/dbt-Athena-orange)
![Airflow](https://img.shields.io/badge/Airflow-2.x-lightblue)
![AWS](https://img.shields.io/badge/AWS-S3%20%7C%20Glue%20%7C%20Athena-yellow)
![Terraform](https://img.shields.io/badge/IaC-Terraform-purple)
![Streamlit](https://img.shields.io/badge/UI-Streamlit-red)

---

## What it does

Binance shows you your orders — it does not tell you if you are profitable.

This pipeline ingests your full Binance Spot trading history (orders, trades, account snapshots) into AWS, transforms the raw data with dbt, and surfaces analytics that Binance does not provide: realized P&L per trade, win rate, asset exposure over time, and commission costs. The Streamlit UI lets you place new trades and explore your performance data in one place.

---

## Architecture

```
Binance API (Testnet / Production)
           │
           ▼
   Python Ingestion Layer
   (pagination, rate limiting)
           │
           ▼
      AWS S3 (raw zone)
      JSON → Parquet
           │
           ▼
   AWS Glue Crawler
   (schema discovery)
           │
           ▼
     AWS Athena
           │
           ▼
   dbt Transformations
   staging → marts
           │
           ▼
   Streamlit UI
   (Streamlit Cloud)
```

Orchestrated locally via **Apache Airflow** (docker-compose). Production deployment config (ECS + EC2) available in `infra/optional/`.

---

## Tech Stack

| Layer          | Tool                        |
|----------------|-----------------------------|
| Ingestion      | Python + python-binance SDK |
| Storage        | AWS S3                      |
| Catalog        | AWS Glue                    |
| Query Engine   | AWS Athena                  |
| Transformation | dbt (Athena adapter)        |
| Orchestration  | Apache Airflow              |
| UI             | Streamlit                   |
| Infrastructure | Terraform                   |

---

## Project Specs

All components are defined in [`specs/`](specs/) before implementation:

| Spec | Description |
|------|-------------|
| [00-overview](specs/00-overview.md) | Goals, stack, constraints, success criteria |
| [01-architecture](specs/01-architecture.md) | System diagram, data flow, AWS services |
| [02-data-model](specs/02-data-model.md) | S3 structure, schemas, Athena tables |
| [03-ingestion](specs/03-ingestion.md) | Binance API → S3, pagination strategy |
| [04-transformation](specs/04-transformation.md) | dbt models, P&L logic, marts |
| [05-orchestration](specs/05-orchestration.md) | Airflow DAGs, triggers, dependencies |
| [06-ui](specs/06-ui.md) | Streamlit pages, components, user flows |
| [07-infrastructure](specs/07-infrastructure.md) | Terraform resources, IAM, naming |

---

## Quick Start

### Prerequisites

- Python 3.12
- Docker + Docker Compose
- AWS account with credentials configured (`aws configure`)
- Terraform >= 1.6
- Binance Testnet API keys → [testnet.binance.vision](https://testnet.binance.vision)

### 1. Clone and install

```bash
git clone git@github.com:viniciusgalvaoia/binance-analytics-pipeline.git
cd binance-analytics-pipeline
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env.example .env
# Fill in: BINANCE_API_KEY, BINANCE_API_SECRET, BINANCE_ENV, AWS credentials
```

### 3. Provision AWS infrastructure

```bash
cd infra/main
terraform init
terraform apply
```

### 4. Start Airflow locally

```bash
docker-compose up -d
# Access Airflow UI at http://localhost:8080
```

### 5. Run the Streamlit app

```bash
streamlit run app/main.py
```

---

## Repo Structure

```
binance-analytics-pipeline/
├── specs/                  # Specs written before code — source of truth
├── infra/
│   ├── main/               # Terraform — actively applied (S3, Glue, Athena, IAM)
│   └── optional/           # Terraform — production deploy (ECS, Kinesis, RDS)
├── ingestion/              # Python: Binance API → S3
├── dbt/                    # dbt project (Athena adapter)
├── airflow/
│   ├── dags/               # DAG definitions
│   └── docker-compose.yml
└── app/                    # Streamlit UI
    └── pages/
```

---

## Cost

~$3–5/month on AWS (S3 storage + Glue crawls + Athena queries).
No always-on compute. Airflow runs locally.

---

## Roadmap

- [ ] Migrate Airflow to ECS (config in `infra/optional/`)
- [ ] Add Kinesis for real-time order streaming
- [ ] Add RDS MySQL for transactional storage
- [ ] Add Redshift for large-scale analytics
- [ ] Multi-user support with API key vault
