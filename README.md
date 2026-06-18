# ⚽ WorldCup Predictor

> A transparent, end-to-end football match prediction pipeline built on BigQuery, Kafka, Python, and Looker/Tableau. No black boxes — every number has an explanation.

---

## What this does

This project predicts World Cup and international match outcomes using two complementary models:

- **1X2 classifier** — multinomial logistic regression estimating win / draw / loss probabilities
- **Scoreline model** — Poisson distribution + 10,000-run Monte Carlo simulation per match

Features are computed entirely in BigQuery SQL (rolling form, head-to-head record, tournament weighting, neutral ground flag). The Python layer only trains, predicts, and writes results back. Everything runs in Docker.

---

## Pipeline overview

```
Kaggle CSVs
    │
    ▼
load_historical.py          ← Phase 1: one-time bulk load
    │
    ▼
BigQuery (raw)              ← results + goalscorers tables
    │
    ▼
BigQuery (features)         ← SQL views: rolling form, H2H, weights
    │
    ▼
Python in Docker            ← Poisson + logistic regression + Monte Carlo
    │
    ▼
BigQuery (predictions)      ← one row per upcoming match
    │
    ▼
Looker / Tableau            ← dashboard with probabilities + scorelines
```

In Phase 2, a **Kafka producer** replays historical matches row by row to simulate a live feed, and the consumer writes to BigQuery in real time.

---

## Stack

| Layer | Tool |
|---|---|
| Data warehouse | BigQuery |
| Feature engineering | SQL (BigQuery views) |
| Modeling | Python — `statsmodels`, `scikit-learn`, `numpy` |
| Containerization | Docker + docker-compose |
| Streaming (Phase 2) | Apache Kafka |
| Dashboard | Looker or Tableau |
| Data source | [Kaggle — International Football Results 1872–2024](https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017) |

---

## Project structure

```
worldcup-predictor/
│
├── data/
│   ├── raw/                    # Kaggle CSVs — see data/raw/README.md
│   └── schemas/                # BigQuery schema JSON files
│
├── ingestion/
│   ├── kafka/
│   │   ├── producer.py         # Replays matches row by row
│   │   └── consumer.py         # Writes to BigQuery
│   └── load_historical.py      # One-time bulk load for Phase 1
│
├── sql/
│   ├── ddl/                    # CREATE TABLE statements
│   │   └── matches.sql
│   └── features/               # BigQuery views for feature engineering
│       ├── rolling_form.sql
│       ├── head_to_head.sql
│       └── tournament_weights.sql
│
├── modeling/
│   ├── poisson.py              # λ estimation + Monte Carlo simulation
│   ├── logistic.py             # 1X2 multinomial classifier
│   ├── predict.py              # Runs both models, writes to BigQuery
│   └── evaluate.py             # Calibration checks, Brier score
│
├── dashboard/
│   └── looker/                 # LookML files or Tableau workbook
│
├── docker/
│   ├── Dockerfile.modeling
│   ├── Dockerfile.kafka
│   └── docker-compose.yml
│
├── notebooks/
│   └── exploration.ipynb
│
├── tests/
│   ├── test_poisson.py
│   └── test_features.py
│
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

---

## Quickstart

### 1. Clone and configure

```bash
git clone https://github.com/your-username/worldcup-predictor.git
cd worldcup-predictor
cp .env.example .env
# Edit .env with your BigQuery project ID and credentials path
```

### 2. Get the data

Download both CSVs from [Kaggle](https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017) and place them in `data/raw/`:

```
data/raw/results.csv
data/raw/goalscorers.csv
```

### 3. Load into BigQuery

```bash
docker-compose run modeling python ingestion/load_historical.py
```

### 4. Run the model

```bash
docker-compose run modeling python modeling/predict.py
```

### 5. Open the dashboard

Connect Looker or Tableau to your BigQuery `predictions` table and open the workbook in `dashboard/`.

---

## Environment variables

Copy `.env.example` to `.env` and fill in your values:

```bash
BQ_PROJECT_ID=your-gcp-project-id
BQ_DATASET=worldcup
GOOGLE_APPLICATION_CREDENTIALS=/app/credentials/keyfile.json
KAFKA_BOOTSTRAP_SERVERS=localhost:9092   # Phase 2 only
```

Never commit `.env` or your keyfile. Both are in `.gitignore`.

---

## Modeling approach

### 1X2 — win / draw / loss

Multinomial logistic regression trained on tournament-weighted historical results. Coefficients are directly interpretable — no SHAP needed. Features include rolling attack/defense rating, head-to-head record, and a neutral ground flag (critical for World Cup since there is no true home team).

### Scoreline — Poisson + Monte Carlo

Each team's expected goals (λ) is estimated from their attack strength vs. the opponent's defense strength, adjusted for recent form. Each match is then simulated 10,000 times. The projected scoreline is the mode of those simulations; the full distribution is stored so the dashboard can show uncertainty, not just a single number.

### Why both?

The 1X2 model is fast and calibrated — good for showing probabilities. The Poisson model is more expressive — it gives you the full scoreline distribution, expected goals, and a sense of how "open" a game is likely to be.

---

## Phases

| Phase | What you build | New skill practiced |
|---|---|---|
| 1 — Batch | CSV → BigQuery → Python → Dashboard | BigQuery, SQL feature engineering, Docker |
| 2 — Simulated streaming | Add Kafka producer/consumer | Kafka, real-time ingestion patterns |
| 3 — Live feed (optional) | Swap Kafka source to API-Football | REST APIs, event-driven architecture |

---

## Data source

**International Football Results 1872–2024** by Mart Jürisoo  
[kaggle.com/datasets/martj42](https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017)

Filtered to: FIFA World Cup, World Cup qualification (UEFA / CONMEBOL), Copa América, and Friendlies (down-weighted).

---

## Contributing

Pull requests welcome. Open an issue first if you're planning something big.

---

## License

MIT
