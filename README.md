# Chatbot Conversations Data Pipeline

---
Demo Link: https://drive.google.com/file/d/1dcSG6Dl8KhiUmHYf52htCuO1TRDWzo_1/view?usp=sharing
## What This Project Does
This pipeline takes a raw chatbot conversation dataset (3,725 question-answer pairs) 
and loads it into a PostgreSQL database automatically using Apache Airflow — 
all running inside Docker so anyone can reproduce it with one command.

The dataset was sourced from Kaggle and simulates real human dialogue, 
making it useful for chatbot training and NLP applications.

---

## Dataset
- **Name:** 3K Conversations Dataset for ChatBot
- **Source:** Kaggle
- **Rows:** 3,725
- **Columns:** id, question, answer
- **Location:** `extraction/Conversation Dataset for Chatbot/sample_data/`
- **Target Table:** `public.chatbot_conversations`

---

## Tech Stack
| Tool | Purpose |
|------|---------|
| Apache Airflow 2.9.3 | Pipeline orchestration |
| PostgreSQL 13 | Target database |
| Docker + Docker Compose | Containerized environment |
| Python + Pandas | Data processing |
| SQLAlchemy | Database connection |
| PyYAML | Schema validation |

---

## Project Structure
extraction/Conversation Dataset for Chatbot/  
├── Subtask4 Documentation & Runbook  
    └── READ.ME  
├── MANIFEST.md                          # Dataset description and target table  
├── .env.sample                          # Credentials template — copy this to .env  
├── .env.example  
├── config/  
│   ├── schema_expected.yaml             # Schema contract — columns, types, nullability  
│   └── create_table.sql                 # PostgreSQL DDL  
├── sample_data/  
│   └── 3K Conversations Dataset for ChatBot.csv   # Source CSV  
├── dags/  
│   └── chatbot_conversations_ingest.py  # Airflow DAG  
├── logs/                                # Airflow task logs  
└── README.md                            # This file  

---

## How to Run

### Prerequisites
- Docker Desktop installed and running
- Git

### Step 1 — Clone the repo
```bash
git clone https://github.com/glynac/glynac-DHAP-34.git
cd glynac-DHAP-34
```

### Step 2 — Set up environment variables
```bash
cp extraction/"Conversation Dataset for Chatbot"/.env.sample .env
```
Edit `.env` and fill in your PostgreSQL credentials if using external PG.
For local dev the defaults work as-is.

### Step 3 — Start the environment
```bash
echo "AIRFLOW_UID=50000" > .env
docker compose up -d
```
Wait 2 minutes for all services to initialize.

### Step 4 — Open Airflow UI
http://localhost:8081

- Username: `airflow`
- Password: `airflow`

### Step 5 — Trigger the pipeline
1. Find `chatbot_conversations_ingest` in the DAG list
2. Toggle it **ON**
3. Click ▶ to trigger manually

### Step 6 — Verify data loaded
```bash
docker exec -it glynac-dhap-34-postgres-1 psql -U airflow -c \
"SELECT COUNT(*) FROM public.chatbot_conversations;"
```
Expected result: **3725 rows**

### Step 7 — Stop the environment
```bash
docker compose down
```

---

## DAG Overview

### Pipeline Flow
file_check → validate_schema → transform → load_to_postgres

### Task Breakdown
| Task | What It Does | Fails If |
|------|-------------|----------|
| file_check | Confirms CSV exists at expected path | File is missing |
| validate_schema | Compares CSV columns against schema_expected.yaml | Column mismatch or unexpected nulls |
| transform | Strips whitespace, drops null rows, saves cleaned CSV | — |
| load_to_postgres | Inserts new rows into PostgreSQL, skips duplicates | DB connection fails |

### Schedule
Runs `@daily` — or trigger manually anytime from the Airflow UI.

---

## Environment Variables
| Variable | Description | Default |
|----------|-------------|---------|
| EXT_PG_HOST | PostgreSQL host | postgres |
| EXT_PG_PORT | PostgreSQL port | 5432 |
| EXT_PG_DB | Database name | airflow |
| EXT_PG_USER | Database user | airflow |
| EXT_PG_PASSWORD | Database password | airflow |
| EXT_PG_SSLMODE | SSL mode | prefer |
| AIRFLOW_UID | Airflow user ID | 50000 |

---

## Results
After a successful run:
- ✅ 3,725 rows loaded into `public.chatbot_conversations`
- ✅ All 4 Airflow tasks completed with green status
- ✅ Schema validated against contract before loading
- ✅ Duplicate rows automatically skipped on rerun

---

## Troubleshooting

**DAG not showing in Airflow UI**
Wait 30 seconds for scheduler to pick it up. Check for syntax errors:
```bash
docker logs glynac-dhap-34-airflow-scheduler-1 --tail 20
```

**file_check fails**
Make sure CSV is in `extraction/Conversation Dataset for Chatbot/sample_data/`

**Permission error on startup**
```bash
docker compose down
echo "AIRFLOW_UID=50000" > .env
docker compose up -d
```

**Schema mismatch error**
Check that CSV has exactly these columns: `id`, `question`, `answer`

**PostgreSQL connection failed**
Check your `.env` file has correct `EXT_PG_*` values and Docker is running.

---

## Runbook

### Rerun with new CSV
1. Replace CSV in `sample_data/`
2. Airflow UI → DAG → Clear task instances
3. Trigger manually

### Update schema when dataset changes
1. Update `config/schema_expected.yaml`
2. Update `config/create_table.sql`
3. Update transform logic in DAG if needed
4. Push and redeploy

### Pre-commit checklist
- [ ] CSV in sample_data/
- [ ] schema_expected.yaml matches CSV columns
- [ ] create_table.sql matches schema
- [ ] No real credentials in .env.sample
- [ ] DAG tested end-to-end
- [ ] README updated
