# Changelog

## v1.0.0

Coursework deliverable. Frozen at completion. Not maintained.

- Fine-tuned DistilBERT on SST-5 for 5-class sentiment. Mapped to 1–5 stars.
- Served the model in-JVM via DJL inside a Kotlin/Ktor service.
- Built three Ktor services: frontend, analyzer, collector.
- Added an async batch path with RabbitMQ and MongoDB.
- Wired up Prometheus and Grafana for service metrics.
- Added unit tests per component and a cross-service integration test.
- Containerized the whole stack. One `docker-compose up` starts everything.
- Deployed to Google Cloud during the course.
