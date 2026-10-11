# Zephract

Multi-tenant AI document processing platform for SMBs: ingest documents (upload, email, SFTP, API), run them through a 5-stage AI pipeline (extraction, classification, entity extraction, enrichment, validation), and search them with keyword, semantic and RAG-powered assistant queries. Verticals are configured through JSON templates.

Stack: Java 21, Spring Boot, PostgreSQL + pgvector, Apache Solr, Amazon Bedrock, SQS, ECS Fargate, S3, Cognito, React + TypeScript, AWS CDK.

## Local development

The `local` environment (architecture spine AD-27) runs every backing service in Docker Compose. No AWS account is needed: floci emulates S3, SQS, SNS, EventBridge and SES.

Prerequisites: Docker Engine with Compose v2, and JDK 21 for the Gradle build.

```bash
cp .env.example .env        # git-ignored; replace the placeholder passwords
docker compose up --wait    # returns once every backing service is healthy
./gradlew spotlessApply check
```

To run `app-api` and the pipeline services on the stack as well (profile `app`; on Windows use `gradlew.bat` and `^` instead of `\` for line breaks, or put the command on one line):

```bash
# builds every service image the `app` profile runs, each as zephract/<service>:<git SHA> and :local
./gradlew :services:app-api:jibDockerBuild :services:intake:jibDockerBuild \
  :services:stage-extraction:jibDockerBuild :services:stage-classification:jibDockerBuild \
  :services:stage-entities:jibDockerBuild :services:stage-enrichment:jibDockerBuild \
  :services:stage-validation:jibDockerBuild :services:worker:jibDockerBuild
docker compose --profile app up --wait       # also runs migrate, then app-api once migrate exits 0
curl http://127.0.0.1:8080/actuator/health/readiness
```

| Service | Host address | Notes |
|---|---|---|
| `postgres` | `127.0.0.1:5432` | Postgres 18 + pgvector. Database `zephract`, owned by role `zephract_migrator` (password `ZEPHRACT_MIGRATOR_PASSWORD`); superuser `postgres` (password `POSTGRES_PASSWORD`). `vector` is installed. |
| `solr` | `http://127.0.0.1:8983/solr` | SolrCloud with embedded ZooKeeper; no collection yet. |
| `valkey` | `127.0.0.1:6379` | Cache only, nothing persisted. |
| `floci` | `http://127.0.0.1:4566` | AWS emulator; region `us-east-1`, credentials `test`/`test`; in-memory, so it starts empty. |
| `app-api` (profile `app`) | `http://127.0.0.1:8080` | Connects as `zephract_app`. Without a token only `/actuator/health` (with `/liveness` and `/readiness`), `/actuator/info`, `/api/v1/openapi.json` and `POST /api/v1/signups` answer; see "Calling the API locally" for a token. Readiness turns DOWN within seconds when Postgres is unreachable. |
| `intake` (profile `app`) | `http://127.0.0.1:8081` | Reads the documents bucket's arrivals from floci's intake queue as `zephract_app`; only health and info are exposed. |
| `stage-extraction` (profile `app`) | `http://127.0.0.1:8082` | Reads EXTRACTION commands from floci's extraction queue as `zephract_app` and writes each run's extracted text; only health and info are exposed. |
| `stage-classification` (profile `app`) | `http://127.0.0.1:8083` | Reads CLASSIFICATION commands from floci's classification queue as `zephract_app` and records each run's Category; the model is the replay fake, which answers Other for any Document it has no fixture for. Only health and info are exposed. |
| `stage-entities` (profile `app`) | `http://127.0.0.1:8084` | Reads ENTITIES commands from floci's entities queue as `zephract_app` and records each run's Entity values; the model is the replay fake, which answers every field missing for any Document it has no fixture for. Only health and info are exposed. |
| `stage-enrichment` (profile `app`) | `http://127.0.0.1:8085` | Reads ENRICHMENT commands from floci's enrichment queue as `zephract_app`, links each run's Entities to the Tenant's Linked Records (flagging an unknown one, or creating it where the template says so), gives a Document without a Client the one Client of its records, flags a duplicate of an earlier Document, and chunks and embeds its masked text (the replay fake is the embedding model locally). Only health and info are exposed. |
| `stage-validation` (profile `app`) | `http://127.0.0.1:8086` | Reads VALIDATION commands from floci's validation queue as `zephract_app`, runs the Category's validation rules over the effective Entities and linked records, records a result per rule, raises RULE_FAILED for a failed `flag` rule and sets the final status: PROCESSED, or REVIEW_NEEDED when any Stage flagged the Document. It calls no model. Only health and info are exposed. |

- Ports bind to `127.0.0.1` only. If a port is taken, set `POSTGRES_PORT`, `SOLR_PORT`, `VALKEY_PORT`, `FLOCI_PORT`, `APP_API_PORT`, `INTAKE_PORT`, `STAGE_EXTRACTION_PORT`, `STAGE_CLASSIFICATION_PORT`, `STAGE_ENTITIES_PORT`, `STAGE_ENRICHMENT_PORT` or `STAGE_VALIDATION_PORT` in `.env`.
- `docker compose down` stops the stack and keeps the Postgres and Solr volumes; `docker compose down -v` also deletes them, and the Postgres init script (`compose/postgres/init/`) runs again on the next start.
- `migrate` is a one-off: Flyway applies `platform/migrations` as `zephract_migrator`, which owns every migrated object, then exits 0; running it again is a no-op apart from re-applying the passwords. V1 creates the runtime roles `zephract_app`, `zephract_router` and `zephract_relay`, whose passwords come from `ZEPHRACT_APP_PASSWORD`, `ZEPHRACT_ROUTER_PASSWORD` and `ZEPHRACT_RELAY_PASSWORD` in `.env` (a Compose-only Flyway callback in `compose/flyway/` sets them). `docker compose --profile app up` runs it before `app-api`; `docker compose run --rm migrate` runs it on its own.
- `app-api`, `intake`, `stage-extraction`, `stage-classification`, `stage-entities`, `stage-enrichment` and `stage-validation` are Jib images on the Temurin 21 JRE base pinned in `compose/docker-compose.images.yml` (entry `jre`), running as a non-root user with a read-only root filesystem. Rebuild them with `jibDockerBuild` after changing the code; Compose never pulls them.
- Images are pinned by tag and digest in `compose/docker-compose.images.yml`. The Testcontainers fixtures in `libs/test-support` read the same file, so tests and Compose run the same images. Change a version there, never in `docker-compose.yml`.
- Dependabot opens one grouped weekly PR for that file, `jre` included, moving tag and digest together. To refresh a pin by hand, for example the `jre` base after a Temurin security release: `docker buildx imagetools inspect eclipse-temurin:<tag>` prints the multi-arch index digest; put `<tag>@<digest>` in the file, then rebuild with `jibDockerBuild` and run the Compose smoke below.

### Calling the API locally (Postman, curl)

Cognito arrives with Epic 8. Until then `tools/local-identity` is the local sign-in: it signs a Cognito-shaped ID token for a user who signed up on the local app-api, with an RSA key generated on your machine the first time it runs (`compose/local-identity/signing-key.json`, git-ignored). app-api trusts the key's public half through `compose/local-identity/jwks/`, mounted read-only. It is a developer tool and is never deployed.

1. Start the stack with `docker compose --profile app up --wait` (images built as above).
2. Sign up, no token needed: `POST http://127.0.0.1:8080/api/v1/signups` with a JSON body such as `{"businessName": "Acme Rentals", "name": "Ada Owner", "email": "ada@example.test", "password": "a-long-password", "verticalKey": "rental", "acceptedTermsVersion": "2026-10"}`. The `201` answer is `{"tenantId": "<id>"}`.
3. Get a token for that user (`gradlew.bat` on Windows): `./gradlew :tools:local-identity:token -Pemail=ada@example.test -Ptenant=<id>`. It reads `.env` for the Postgres port and `ZEPHRACT_APP_PASSWORD`, and prints a token valid for 8 hours.
4. On the very first run the key is new, so run `docker compose restart app-api` once to make app-api load it. Later tokens need no restart.
5. In Postman, set Authorization to Bearer Token with the printed value, and call for example `GET http://127.0.0.1:8080/api/v1/users`. Postman can import every endpoint from `http://127.0.0.1:8080/api/v1/openapi.json`.

Locally the models are the replay fake (AD-27): uploads go through the whole pipeline, but with no recorded answer for your files classification says Other with confidence 0, entities come back missing, and the Assistant has no recorded answers to give. Real answers need Bedrock (Epic 8).

### CI smoke

The Build workflow's `Compose smoke` job runs the same steps on every PR and push to `main`. To run it locally on a clean stack (it refuses to overwrite an existing `.env`):

```bash
scripts/ci/env-from-example.sh   # .env from .env.example with generated passwords
scripts/ci/compose-smoke.sh      # up --wait, jibDockerBuild, --profile app up --wait, readiness UP
docker compose --profile app down -v
```
