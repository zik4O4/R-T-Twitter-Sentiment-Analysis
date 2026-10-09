# Streaming Sentiment Analysis

A Python sentiment-analysis pipeline that replays CSV records through Kafka, classifies text with PySpark ML, stores predictions in MongoDB and presents them in a Django dashboard.

## Problem and solution

The project connects text classification with message processing and a web interface. The producer sends a validation record every three seconds; the consumer applies a saved Spark model and persists the result. This is CSV replay, not live Twitter/X API ingestion.

## Architecture

| Component | Responsibility |
|---|---|
| `ML PySpark Model/Big_Data.ipynb` | Train and evaluate a Tokenizer → StopWordsRemover → CountVectorizer → LogisticRegression pipeline |
| `Kafka-PySpark/producer-validation-tweets.py` | Publish CSV records to Kafka topic `numtest` |
| `Kafka-PySpark/consumer-pyspark.py` | Clean text, run Spark inference and write to `bigdata_project.tweets` in MongoDB |
| `Django-Dashboard/` | Show sentiment distributions and classify submitted text |
| `zk-single-kafka-single.yml` | Provision Kafka and ZooKeeper for local development |

**Technologies:** Python, PySpark, Kafka, MongoDB, Django, pandas, NLTK, Matplotlib, Seaborn and Docker Compose.

## Local setup

Use an isolated Python environment compatible with the pinned packages, a Java runtime supported by PySpark 3.5.1, Docker Compose, and MongoDB at `localhost:27017`. Python 3.10 or 3.11 is a reasonable starting point; the entire service stack has not been independently reproduced.

```bash
git clone https://github.com/zik4O4/R-T-Twitter-Sentiment-Analysis.git
cd R-T-Twitter-Sentiment-Analysis
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
docker compose -f zk-single-kafka-single.yml up -d
```

On Windows, activate the environment with `.venv\Scripts\Activate.ps1` in PowerShell. Start a separate local MongoDB instance; the Compose file does not include MongoDB or Django. Ensure Kafka topic `numtest` exists, or that your local broker permits automatic topic creation.

The existing requirements specify the legacy `kafka==1.3.5` distribution. Client compatibility with your interpreter must be checked before running; do not install a second distribution that exposes the same `kafka` namespace in this environment. The training notebook also needs `jupyter` and `findspark`:

```bash
python -m pip install jupyter findspark
```

NLTK downloads are initiated by the source and require network access. The supplied Spark model directories are named `logistic_regression_model.pkl`, but contain Spark metadata and Parquet files rather than a single Python pickle. Use the model saved with a compatible Spark version.

## Run the pipeline

In separate activated terminals, starting from the repository root:

```bash
# Terminal 1
cd Kafka-PySpark
python consumer-pyspark.py
```

```bash
# Terminal 2
cd Kafka-PySpark
python producer-validation-tweets.py
```

```bash
# Terminal 3: keep this generated key local and stable between restarts
export DJANGO_SECRET_KEY="$(python -c 'import secrets; print(secrets.token_urlsafe(50))')"
cd Django-Dashboard
python manage.py migrate
python manage.py runserver
```

Open `http://127.0.0.1:8000`. Configure `PYSPARK_PYTHON` and `PYSPARK_DRIVER_PYTHON` yourself if your Spark setup requires explicit interpreter paths. Database and model paths remain local-development defaults.

To inspect training, launch `jupyter notebook "ML PySpark Model/Big_Data.ipynb"` and execute from its dataset directory. The notebook records validation accuracy **0.878** for the supplied sentiment dataset and labels Negative, Positive, Neutral and Irrelevant. This is a saved historical output, not a new benchmark.

## Data and results

Training and validation CSVs and saved model copies are included. Their upstream source and redistribution terms still need an explicit citation. Existing figures are under [`imgs/`](imgs/); they should be interpreted alongside the source, not as evidence of a currently running deployment.

## Limitations and configuration

- The consumer processes records one at a time; this is not a demonstrated Spark Structured Streaming deployment.
- Text cleaning and label mapping are duplicated between training and inference and should be kept aligned.
- The dashboard creates a Spark session when its classification module is imported, so Java and model files must be available.
- Set a fresh `DJANGO_SECRET_KEY` locally. Previously published keys must be rotated if used by a deployment; a code change does not invalidate old keys.
- `DJANGO_DEBUG` defaults to local development mode. Production requires an explicit secret and allowed hosts, plus deployment hardening.
- Historical bytecode, SQLite files and collected static assets remain in the repository; new ignore rules do not remove tracked files.

## License and attribution

No repository-level license has been selected. Dataset and third-party component terms must be checked separately. Existing project and contributor attribution is preserved in the source.
