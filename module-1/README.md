# 🚕 NYC Taxi Data Ingestion Pipeline

![Python](https://img.shields.io/badge/python-3.13-blue.svg)
![Docker](https://img.shields.io/badge/docker-compose-2496ED.svg)
![PostgreSQL](https://img.shields.io/badge/postgresql-18-336791.svg)

An automated ETL pipeline designed to fetch, process, and load NYC Taxi Trip data into a PostgreSQL database using Docker orchestration.

## 📋 Overview
This project automates the data engineering workflow for the NYC Taxi dataset. It handles the full lifecycle from data retrieval to storage, ensuring environmental consistency through containerization.

- **Data Source**: NYC TLC Trip Record Data.
- **Infrastructure**: Multi-container architecture with health-check synchronization.
- **Processing**: Chunked data loading using Pandas to optimize memory usage.

## 🏗️ Architecture
The system consists of two primary services:
1. **`pgdatabase`**: A PostgreSQL 18 instance with persistent volume mapping .
2. **`ingest_data`**: A Python-based worker that executes the ETL logic (Download -> Chunking -> Loading).

## 🚀 Quick Start

### Prerequisites
- Docker Desktop installed and running.
- A SQL client like **DBeaver**.
---
