# Simplon RAG Sample

<!-- markdownlint-disable -->
<p align="center">
  <strong>Sample RAG support chatbot — powered by RAG, LangChain, and Mistral</strong>
</p>

<p align="center">
  <a href="https://opensource.org/licenses/MIT">
    <img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License: MIT" />
  </a>
  <a href="https://python-semantic-release.readthedocs.io/">
    <img src="https://img.shields.io/badge/semantic--release-python-e10079?logo=semantic-release" alt="semantic-release: python" />
  </a>
</p>
<!-- markdownlint-restore -->

## Déploiement GCP (Google Cloud Platform)

Cette section détaille l'infrastructure Cloud sous-jacente, la gestion stricte des privilèges (principe du moindre privilège) ainsi que les procédures d'exploitation de l'application Simplon RAG.

### 1. Services GCP utilisés

| Service GCP | Rôle dans l'architecture |
| :--- | :--- |
| **Cloud Run** | Héberge et exécute de manière serverless les conteneurs du Frontend (Streamlit) et de l'API (FastAPI). |
| **Cloud SQL (PostgreSQL)** | Stocke de manière managée les données relationnelles et utilise l'extension `pgvector` pour les embeddings du RAG. |
| **Artifact Registry** | Stocke de manière sécurisée les images de conteneurs Docker poussées pour le déploiement. |
| **Cloud Storage** | Stocke les documents bruts (corpus de documents PDF/TXT) importés pour alimenter le système RAG. |
| **Secret Manager** | Centralise et sécurise les clés d'API (Mistral AI, etc.) pour éviter de les exposer en clair. |

---

### 2. Matrice des Rôles IAM et Comptes de Service (SA)

Afin de respecter le **principe du moindre privilège**, chaque composant possède son propre compte de service avec des permissions minimales et ciblées.

#### A. GitHub Actions CI/CD (`sa-github-actions@...`)
* **Rôles attribués :**
    * `roles/artifactregistry.writer` (Administrateur d'Artifact Registry)
    * `roles/run.admin` (Administrateur Cloud Run)
    * `roles/iam.serviceAccountUser` (Utilisateur de compte de service)
* **Justification :** Ces permissions permettent uniquement d'injecter une nouvelle image Docker dans le registre et de déclencher une mise à jour des révisions Cloud Run. Le rôle `serviceAccountUser` est requis pour que GitHub puisse lier les services Cloud Run à leurs comptes de service respectifs sans posséder de permissions globales d'administration du projet.

#### B. API FastAPI (`sa-rag-api@...`)
* **Rôles attribués :**
    * `roles/cloudsql.client` (Client Cloud SQL)
    * `roles/storage.objectViewer` (Lecteur des objets Storage)
    * `roles/secretmanager.secretAccessor` (Accesseur de secrets)
* **Justification :** L'API doit pouvoir joindre la base de données via le proxy Cloud SQL, lire les documents à vectoriser dans le bucket, et décoder les clés d'API privées (Mistral AI) au démarrage. Elle n'a aucun droit d'écriture sur le bucket ni de modification sur l'infrastructure (pas de droits `admin` ou `editor`).

#### C. Frontend Streamlit (`sa-rag-frontend@...`)
* **Rôles attribués :**
    * *(Aucun rôle GCP spécifique requis)*
* **Justification :** Le Frontend est une interface publique isolée. Elle communique uniquement avec le Back via des requêtes HTTP standard (URL publique de l'API). Elle n'interagit directement avec aucun service interne de l'infrastructure GCP, éliminant ainsi toute surface d'attaque en cas de compromission du Front.

---

### 3. Procédure de Rollback d'une révision Cloud Run

En cas d'anomalie détectée en production après un déploiement, Cloud Run permet un retour en arrière (rollback) instantané et sans coupure de service (zéro-downtime) grâce à l'historique des révisions.

1. Accédez à la console **Cloud Run** et sélectionnez le service concerné (`simplon-rag-api` ou `simplon-rag-frontend`).
2. Allez sur l'onglet **Révisions**.
3. Identifiez la dernière version stable connue dans la liste (basée sur la date ou le tag de version).
4. Cliquez sur les trois petits points verticaux `...` à l'extrémité droite de la ligne de cette révision stable, puis sélectionnez **Gérer le trafic**.
5. Modifiez l'attribution du trafic en passant la version défaillante à `0%` et attribuez `100%` du trafic à la révision stable sélectionnée.
6. Cliquez sur **Enregistrer**. La bascule réseau est immédiate.

---

### 4. URLs de l'Environnement de Production

* **API (FastAPI Backend) :** `https://simplon-rag-api-572169748454.europe-west9.run.app`
* **Interface Utilisateur (Streamlit Frontend) :** `https://simplon-rag-frontend-572169748454.europe-west9.run.app`

---

Intelligent support chatbot example, built on a Retrieval-Augmented Generation (RAG) architecture
using LangChain, LangGraph, PostgreSQL/pgvector for vector storage, and Mistral for both
embeddings and LLM inference.

## Features

- **Document Ingestion** - PDF upload with SHA-256 deduplication, chunking, and embedding
- **RAG Pipeline** - Semantic retrieval via pgvector cosine similarity + LLM generation
- **LangGraph Agent** - Stateful multi-step graph: routing, retrieval, generation, history
- **Mistral AI** - `mistral-embed` (1024 dims) for embeddings, `mistral-large-latest` for LLM
- **PostgreSQL + pgvector** - HNSW index for fast approximate nearest-neighbour search
- **FastAPI REST API** - 8 endpoints under `/api/v1` for ingestion, chat, and evaluation
- **Ragas Evaluation** - Faithfulness, answer relevancy, and context recall metrics

## Tech Stack

| Category | Technology |
|----------|------------|
| Language | Python >= 3.14 |
| Package Manager | uv |
| LLM Framework | LangChain + LangGraph |
| LLM / Embeddings | Mistral AI |
| Vector Store | PostgreSQL + pgvector |
| ORM / Migrations | SQLAlchemy (async) + Alembic |
| API | FastAPI + uvicorn |
| RAG Evaluation | Ragas |

## Quickstart with Docker

The fastest way to spin up the full stack (PostgreSQL + API + Streamlit UI):

```bash
# 1. Configure the Mistral API key (required)
cp api/.env.example api/.env
# Edit api/.env and set MISTRAL_API_KEY

# 2. Start everything in development mode (hot reload, source bind mounts)
docker compose up -d

# 3. Open the UI
open http://localhost:8501       # Streamlit chat
# API docs:    http://localhost:8000/docs
# API health:  http://localhost:8000/api/v1/health

# 4. Tear down
docker compose down              # keep data
docker compose down -v           # also drop the postgres volume
```

For a production-like build (multi-worker uvicorn, no source mounts, postgres
port hidden from the host):

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

## Local installation (without Docker)

```bash
# Copy and configure environment
cp api/.env.example api/.env
# Edit api/.env with your API keys and DB connection

# Install API dependencies
cd api
uv sync --extra dev          # dev tools included

# Apply database migrations (requires a running PostgreSQL with pgvector)
uv run alembic upgrade head
cd ..

# Install frontend dependencies
cd frontend
uv sync
cd ..

# Install git hooks
pre-commit install
```

## Usage (local)

```bash
# Run API (from api/)
cd api && uv run python main.py
# API available at http://localhost:8000/api/v1

# Run the Streamlit chat UI (from frontend/)
cd frontend && uv run streamlit run src/app/app.py
# UI available at http://localhost:8501
```

### CLI Tools

Standalone entry points for ingestion and evaluation, runnable without the API
(useful for cron, CI, or one-off scripts). Run from `api/`.

```bash
cd api

# Ingest every PDF in data/docs/ (idempotent via SHA-256)
uv run python -m rag.cli.ingest
uv run python -m rag.cli.ingest --docs-dir path/to/pdfs

# Run Ragas evaluation against data/evaluation/samples.json
uv run python -m rag.cli.eval
uv run python -m rag.cli.eval --samples path/to/samples.json
```

## Development

```bash
# Run API tests (from api/)
cd api && uv run pytest

# Lint all files (from repo root)
uv run pymarkdownlnt scan --recurse .
uv run yamllint .

# Commit (Conventional Commits format)
git commit -m "feat: ..."
```

## Documentation

| File | Description |
|------|-------------|
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | Contribution guidelines |
| [`CHANGELOG.md`](CHANGELOG.md) | Version history |

## License

MIT License - see the [LICENSE](LICENSE) file for details.

## Author

**Maxime Lenne**
