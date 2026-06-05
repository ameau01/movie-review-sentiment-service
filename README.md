# Movie Review Sentiment Service

### Taking a fine-tuned transformer out of the notebook: DistilBERT served behind a containerized, observable microservice stack

**Graduate coursework — [CSCA-5028: Applications of Software Architecture for Big Data](https://www.colorado.edu/cs/academics/online-programs/mscs-coursera/csca5028), M.S. Computer Science, University of Colorado Boulder**

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Kotlin](https://img.shields.io/badge/Kotlin-2.2-purple.svg)](https://kotlinlang.org)
[![Ktor](https://img.shields.io/badge/Ktor-3.3-blue.svg)](https://ktor.io)
[![Docker](https://img.shields.io/badge/Docker-Compose-blue.svg)](https://docker.com)

An end-to-end system that fine-tunes a transformer for fine-grained movie-review sentiment, then **operationalizes it** — containerized serving, metrics instrumentation, async batch processing, persistence, and dashboards, deployed to Google Cloud during development. The emphasis is the **MLOps path around the model**: how a Python-trained model becomes a running, monitored, containerized service inside a JVM stack — not the model's raw accuracy.

---

### What this project is (and isn't)

This is an **ML-serving / MLOps** project, not a modeling research project. The interesting engineering is operationalization: a Hugging Face DistilBERT model, fine-tuned in Python, then loaded and served **in-process inside a Kotlin/Ktor service via DJL (Deep Java Library)** — running the HuggingFace tokenizer and transformer on the JVM rather than behind a separate Python sidecar — wired into a multi-service stack with RabbitMQ, MongoDB, and Prometheus/Grafana, all orchestrated by docker-compose.

To be precise about the observability layer: this monitors **service health and throughput** (request latency, counts, `/health` checks) via Prometheus/Grafana — the infrastructure layer of MLOps. **Model-level monitoring** (prediction drift, eval gates, a model registry, retraining CI/CD) is the documented next layer, not something this project claims to do.

The model is a baseline, not a headline (see Model section below). The point is the operationalization.

---

### Engineering highlights (the MLE-relevant parts)

- **In-JVM transformer inference via DJL** — the fine-tuned DistilBERT and its HuggingFace tokenizer run inside the Kotlin service through Deep Java Library, not behind a separate Python process. One serving runtime, no cross-language network hop on the inference path.
- **Two execution paths from one model** — a low-latency interactive path (web UI → REST → prediction) and an asynchronous batch path (external API → RabbitMQ → consumer → MongoDB), both backed by the same model service.
- **Containerized and cloud-deployed** — the full stack comes up with one `docker-compose` command and was deployed to Google Cloud during development.
- **Service observability built in** — Prometheus scrapes each service; Grafana dashboards over latency, throughput, and health.
- **Tested** — per-component unit tests (api, database, messaging, sentiment) plus a cross-service integration test, run under Gradle/JUnit.

---

### Architecture

A multi-service design coordinating an interactive inference path and an asynchronous batch path:

1. **Frontend Server** (Ktor) — web UI to submit a review and get a 1–5 star rating
2. **Sentiment Analyzer** (Ktor) — loads the fine-tuned DistilBERT model and serves predictions over REST
3. **Data Collector** (Ktor) — pulls critic reviews from an external API and publishes them to the queue
4. **RabbitMQ** — decouples the collector from the analyzer for batch processing
5. **MongoDB** — stores reviews and predicted ratings for reporting (e.g., top-10 movies by sentiment)
6. **Prometheus + Grafana** — service metrics and dashboards

Each service exposes a `/health` endpoint used by Compose healthchecks.

| Layer | Technology |
|---|---|
| ML model | DistilBERT (Hugging Face Transformers), fine-tuned on SST-5 |
| Training | Python, PyTorch, Hugging Face `Trainer` |
| Model serving | DJL (Deep Java Library) — in-JVM inference with the HuggingFace tokenizer |
| Backend services | Kotlin, Ktor (Netty), REST |
| Frontend | FreeMarker HTML templates |
| Messaging | RabbitMQ |
| Persistence | MongoDB |
| Containerization | Docker, docker-compose |
| Cloud | Google Cloud (deployed during development) |
| Observability | Prometheus, Grafana |
| Testing | JUnit — per-component unit tests + cross-service integration test |
| Build | Gradle (Kotlin DSL) |

---

### Model: fine-tuned DistilBERT on SST-5

- **Dataset:** SST-5 (Stanford Sentiment Treebank, 5 classes), ~11,855 phrases
- **Model:** `distilbert-base-uncased` fine-tuned for 5-class sequence classification (3 epochs, lr 2e-5, batch 32)
- **Class → rating mapping:** very negative → 1 star, through very positive → 5 stars
- **Measured results (test set):** accuracy **0.526**, F1 **0.515**

A note on that number: SST-5 fine-grained (5-class) sentiment is a genuinely hard benchmark — strong models cluster in the low-to-mid 50s because adjacent classes (e.g. "negative" vs "very negative") are inherently ambiguous. 52.6% is a reasonable baseline for a fine-tuned DistilBERT and is reported here as-is. Improving it (class-weighted loss, collapsing to 3 classes, a larger backbone) is documented as future work rather than claimed as done.

Training notebook: `docs/Movie_Ratings_Sentiment_Analysis_v8.ipynb`.

---

### Quick start (local)

Fully containerized — one command brings up the model service, frontend, RabbitMQ, MongoDB, and monitoring.

```bash
git clone https://github.com/ameau01/movie-review-sentiment-service.git
cd movie-review-sentiment-service
docker-compose up --build
```

Default ports:

| Service | URL |
|---|---|
| Frontend UI | http://localhost/ |
| Analyzer | http://localhost:8881/ |
| Collector | http://localhost:8882/ |
| RabbitMQ UI | http://localhost:15672/ |
| Prometheus | http://localhost:9090/ |
| Grafana | http://localhost:3000/ |
| MongoDB | mongodb://localhost:27017 |

---

### Repository layout

```
applications/   three Ktor services (analyzer, collector, frontend)
components/      shared modules (api, database, messaging, sentiment)
data/SST-5/      training dataset
docs/            final report, system design, training notebook
integration-test/  cross-service integration tests
scripts/         helper scripts (build-model, run, db/mq examples)
workflows/       deployment script
```

---

### MLOps maturity — honest scope

This is the foundational tier of an MLOps stack — the infrastructure layer. Model-quality and operational-governance layers are documented here as out-of-scope course choices, not hidden gaps.

| Layer | Status | Notes |
|---|---|---|
| Model training pipeline | ✅ | Reproducible from the `docs/` notebook |
| In-JVM model serving (DJL) | ✅ | DistilBERT + HF tokenizer, in-process |
| Containerization | ✅ | docker-compose, multi-service stack |
| Cloud deployment | ✅ | Deployed to GCP during the course |
| Async batch processing | ✅ | RabbitMQ broker |
| Service observability | ✅ | Prometheus + Grafana — latency, throughput, health |
| Unit + integration tests | ✅ | Gradle / JUnit |
| Centralized logging | ⚠️ | Per-container stdout; no ELK/Loki aggregation |
| CI/CD beyond `gradle test` | ⚠️ | Tests run locally; no automated deploy pipeline |
| Model registry + versioning | ❌ | Out of scope — MLflow / a model registry would be the natural addition |
| Model drift monitoring | ❌ | Out of scope — would extend the Prometheus layer with prediction-quality metrics |
| A/B testing + canary deploys | ❌ | Single-instance services |
| Load testing | ❌ | Not performed |

The ❌ rows are deliberately named rather than omitted: they map the path from this foundational tier to a production-grade platform.

---

## Project status

[![Version](https://img.shields.io/badge/version-1.0.0-blue)](https://github.com/ameau01/movie-review-sentiment-service/releases)
[![Status: Complete](https://img.shields.io/badge/status-complete-brightgreen)](https://github.com/ameau01/movie-review-sentiment-service/releases)

Graduate coursework, frozen at v1.0.0. Not under active development.

- Fine-tuned DistilBERT on SST-5. Test accuracy 0.526 / F1 0.515.
- In-JVM model serving via DJL inside a Kotlin/Ktor service.
- Three Ktor services: frontend, analyzer, collector.
- Async batch path: RabbitMQ to MongoDB.
- Service observability: Prometheus and Grafana.
- Unit tests per component, plus a cross-service integration test.
- Full stack runs from one `docker-compose up`. Deployed to GCP during the course.

---

### License

MIT.

---

## Citation

```
@misc{movie_review_sentiment_service_2026,
  title   = {Movie Review Sentiment Service},
  author  = {Alexander Meau},
  year    = {2026},
  version = {1.0.0}
}
```
