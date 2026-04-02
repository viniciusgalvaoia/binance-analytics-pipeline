# 00 — Project Overview

## Intent

Binance's native interface shows balances and open orders but offers no analytical
depth. Traders who want to understand their actual performance — realized P&L per
trade, win rate, asset exposure over time, commission costs — have to export CSVs
manually or use expensive third-party tools.

This project builds a self-hosted analytics pipeline on AWS that ingests trading
data from the Binance API, transforms it with dbt on Athena, and exposes it through
a Streamlit app where users can both place spot trades and explore their full
performance history.

## Goals

- Give spot traders an analytical view of their trading history that Binance does
  not provide natively (P&L, win rate, trade performance, asset exposure)
- Demonstrate a production-grade AWS data pipeline (S3 → Glue → Athena → dbt)
  provisioned entirely with Terraform
- Provide a working UI that users can connect to their own Binance API keys

## Non-Goals

- Futures or margin trading
- Real-time price streaming or order book data
- Automated or algorithmic trading strategies
- Multi-exchange support (Binance Spot only)
- Built-in user authentication (API keys are user-managed via the UI)

## Success Criteria

- [ ] Users can place market and limit spot orders via the Streamlit UI
- [ ] Users can view open orders and cancel them from the UI
- [ ] On-demand pipeline ingests order history, trade history, and account
      snapshots from Binance into S3
- [ ] dbt models calculate realized P&L, win rate, and asset exposure from
      raw trade data
- [ ] UI displays: P&L over time, win rate, top/worst trades, asset exposure,
      commission costs, full trade history table
- [ ] All AWS infrastructure (S3, Glue, Athena, IAM) is provisioned via
      Terraform with no manual steps
- [ ] Pipeline can be triggered on-demand from the UI or locally via Airflow

## Environments

| Environment | Binance Endpoint                   | Purpose                       |
|-------------|------------------------------------|-------------------------------|
| Testnet     | https://testnet.binance.vision/api | Development and demo          |
| Production  | https://api.binance.com            | Real account (future rollout) |

The codebase supports both environments via a single `BINANCE_ENV` config flag.
No code changes required to switch — only credentials and base URL differ.

## Key Data Sources

| Endpoint              | Data                                      | Used For                  |
|-----------------------|-------------------------------------------|---------------------------|
| GET /api/v3/myTrades  | price, qty, quoteQty, commission, isBuyer | P&L calculation, win rate |
| GET /api/v3/allOrders | orderId, status, type, side, time         | Order history, fill rate  |
| GET /api/v3/account   | balances per asset (free + locked)        | Portfolio snapshot        |
| GET /api/v3/ticker    | current price per symbol                  | Unrealized P&L            |

> Binance does not provide P&L data. All P&L metrics are calculated by the dbt
> transformation layer from raw trade history.

> `allOrders` and `myTrades` endpoints are limited to 24-hour windows per request.
> The ingestion layer paginates by date range to retrieve full history.

## Stack

| Layer          | Tool                        | Notes                              |
|----------------|-----------------------------|------------------------------------|
| Ingestion      | Python + python-binance SDK | Handles pagination and rate limits |
| Storage        | AWS S3                      | Raw JSON and processed Parquet     |
| Catalog        | AWS Glue                    | Schema registry for Athena         |
| Query engine   | AWS Athena                  | Serverless SQL on S3               |
| Transformation | dbt (Athena adapter)        | Staging → marts, P&L logic         |
| Orchestration  | Apache Airflow (local)      | docker-compose, on-demand trigger  |
| UI             | Streamlit (Streamlit Cloud) | Trading terminal + dashboards      |
| Infrastructure | Terraform                   | S3, Glue, Athena, IAM              |

## Cost Target

Under $10/month on AWS (S3 + Glue + Athena only).
Airflow runs locally. No RDS, Kinesis, or Redshift in active deployment.

## Roadmap (out of current scope)

- Migrate Airflow to ECS (Terraform config kept in `infra/optional/`)
- Add Kinesis Data Streams for real-time order events
- Add RDS MySQL for transactional order storage
- Add Redshift for analytical queries at scale
- Add multi-user support with API key vault
