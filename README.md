# Best Car Dealership

Hello! Welcome to my capstone project - a car dealership review platform where users can browse dealerships, read customer reviews, and (once logged in) post their own reviews. Each review is automatically scored for sentiment (positive, neutral, or negative) by a sentiment-analysis microservice.

This project was built as the capstone for the IBM Full Stack Software Developer Professional Certificate. It combines a React frontend, a Django backend acting as an API gateway, a Node.js/Express + MongoDB microservice, and a serverless sentiment analyzer. All containerized and deployable to Kubernetes.

## Architecture

The app's made up of several services that each own a piece of the system, with Django as the hub the user actually talks to:

```
                    ┌─────────────────────────────┐
   User ──────────► │   Django app (the hub)      │
                    │   - Serves React frontend   │
                    │   - User auth (SQLite)       │
                    │   - Car make/model data      │
                    │   - Proxies out to services  │
                    └──────────┬──────────┬────────┘
                               │          │
                     ┌─────────▼───┐   ┌──▼──────────────────┐
                     │ Express +   │   │ Sentiment Analyzer  │
                     │ MongoDB     │   │ (IBM Code Engine)   │
                     │ dealers &   │   │ scores each review  │
                     │ reviews     │   │ pos/neu/neg         │
                     └─────────────┘   └─────────────────────┘
```

- **React frontend** — the UI the user sees (login, register, dealer list, dealer detail, review submission). Compiled to static files and served by Django.
- **Django backend** — the central app. Serves the frontend, owns user authentication and the car make/model data (in SQLite), and acts as a proxy/gateway: it calls the other services on the user's behalf and processes their responses. The browser only ever talks to Django.
- **Express + MongoDB microservice** — a standalone Node.js service that stores and serves dealer and review data.
- **Sentiment analyzer** — a separate microservice (NLTK-based) deployed on IBM Code Engine that classifies review text.

## Tech stack

- **Frontend:** React
- **Backend:** Django (Python)
- **Microservice:** Node.js, Express, MongoDB (via Mongoose)
- **Sentiment analysis:** Python (NLTK), deployed on IBM Code Engine
- **CI/CD:** GitHub Actions (automated linting for Python and JavaScript)
- **Containerization / deployment:** Docker, Kubernetes

## Prerequisites

- Python 3.12
- Node.js and npm
- Docker (with Docker running)
- A deployed sentiment-analyzer service (see [Sentiment analyzer](#4-sentiment-analyzer))

## Setup

Clone the repo, then set up each piece. The services can run independently, but the app is only fully functional when the Django app, the Express/Mongo microservice, and the sentiment analyzer are all reachable.

### 1. Django backend

The Django app is the hub — set this up first.

```bash
cd server
pip install virtualenv
virtualenv djangoenv
source djangoenv/bin/activate
python3 -m pip install -U -r requirements.txt
python3 manage.py makemigrations
python3 manage.py migrate
python3 manage.py runserver
```

Runs on **port 8000**. Create a superuser (`python3 manage.py createsuperuser`) if you need admin access to seed car make/model data.

### 2. React frontend

The frontend is a React app that gets compiled into static bundles Django serves.

```bash
cd server/frontend
npm install
npm run build
```

This compiles the React source (`frontend/src/`) into `frontend/build/`. Django serves this build folder — so **any change to React source requires re-running `npm run build`** for it to take effect. Django's `settings.py` references `frontend/build` and `frontend/build/static`.

### 3. Node.js/Express microservice

The dealer/review microservice is an Express app backed by MongoDB, run via Docker Compose (Mongo and Node in separate containers).

```bash
cd server/database
docker build . -t nodeapp
docker-compose up
```

The entry file is `app.js`, and it exposes endpoints like `/fetchDealers`, `/fetchDealer/:id`, `/fetchReviews/dealer/:id`, and `/insert_review`. Runs on **port 3030**. **Re-run `docker build` after any change to `app.js`**, since the image bakes in the code.

### 4. Sentiment analyzer

A separate microservice deployed on IBM Code Engine. The Django app reaches it via an environment variable, not hardcoded.

The deployment source lives in `server/djangoapp/microservices` (a `sentiment_analyzer.py` and a `Dockerfile`). Build, push, and deploy it to Code Engine, then note the resulting URL for the environment configuration below.

## Environment configuration

The Django app reads two service URLs from `server/djangoapp/.env`:

```
backend_url=<your Express/Mongo microservice URL>
sentiment_analyzer_url=<your Code Engine sentiment analyzer URL>
```

> **Cloning this repo?** The committed `.env` points at the original author's deployed services, which won't be reachable for you. Update both `backend_url` and `sentiment_analyzer_url` to point at your own running microservice and deployed analyzer.

## Deployment (Docker + Kubernetes)

The Django app can be containerized and deployed to Kubernetes:

```bash
# Build and push the image (replace with your container registry namespace)
docker build -t <registry>/<namespace>/dealership .
docker push <registry>/<namespace>/dealership

# Deploy to Kubernetes
kubectl apply -f deployment.yaml
kubectl port-forward deployment.apps/dealership 8000:8000
```

The `Dockerfile` runs the app with **gunicorn** (a production WSGI server) rather than the Django dev server, and `entrypoint.sh` runs migrations and `collectstatic` on container startup. `deployment.yaml` defines the Kubernetes deployment. Make sure to update the image path to your own namespace before applying.

> **Note:** the container starts with a fresh database, so you'll need to create a superuser and create your own car make/model data inside the running pod (the dealer and review data comes from the external Mongo service and is unaffected).

## CI/CD

A GitHub Actions workflow (`.github/workflows/main.yml`) automatically lints all Python files (flake8) and JavaScript files (JSHint) on every push and pull request to `main`. Linter configuration lives in `.flake8` and `.jshintrc`.

## Credit

This project was built on a skeleton provided by IBM / Skills Network as part of the Full Stack Software Developer Professional Certificate capstone.

Thanks again for checking out my project! ^.^
