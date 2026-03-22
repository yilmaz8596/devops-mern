# DevOps MERN — Task Manager

A full-stack Task Manager application built with the **MERN** stack (MongoDB, Express, React, Node.js), fully containerized with **Docker** and **Docker Compose**, with a **GitHub Actions** CI/CD pipeline.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite |
| Backend | Node.js, Express |
| Database | MongoDB 7 (containerized) |
| Containerization | Docker, Docker Compose |
| CI/CD | GitHub Actions |
| Deployment | AWS EC2 |
| Orchestration | Kubernetes (kubectl) |

---

## Project Structure

```
devops-mern/
├── client/                  # React frontend (Vite)
│   ├── src/
│   │   ├── components/      # UI components (TaskCard, TaskList, etc.)
│   │   ├── services/        # API calls (taskService.js)
│   │   └── utils/           # Helper utilities
│   └── Dockerfile
├── server/                  # Express backend
│   ├── controllers/         # Route handlers
│   ├── models/              # Mongoose schemas
│   ├── routes/              # API route definitions
│   ├── .env                 # Environment variables (not committed)
│   └── Dockerfile
├── docker-compose.yml       # Multi-container orchestration
├── k8s/                     # Kubernetes manifests
│   ├── mongo-deployment.yml
│   ├── backend-deployment.yml
│   └── client-deployment.yml
└── .github/
    └── workflows/
        └── ci.yml           # GitHub Actions CI/CD pipeline
```

---

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/tasks` | Get all tasks |
| `POST` | `/api/tasks` | Create a new task |
| `DELETE` | `/api/tasks/:id` | Delete a task by ID |
| `GET` | `/health` | Server health check |

---

## Environment Variables

### `server/.env`

```env
PORT=5000
MONGO_URI=mongodb://mongo:27017/devops-mern
```

> **Note:** When running outside Docker (e.g. local `node server.js`), set `MONGO_URI=mongodb://localhost:27017/devops-mern`.

---

## Running with Docker Compose

### Prerequisites
- [Docker](https://docs.docker.com/get-docker/) installed and running

### Start all services

```bash
docker-compose up -d
```

This starts three containers:

| Container | Description | Port |
|---|---|---|
| `mongo` | MongoDB 7 database | `27017` |
| `backend` | Express API server | `5000` |
| `frontend` | React app (Vite preview) | `5173` |

### Stop all services

```bash
docker-compose down
```

### View logs

```bash
docker-compose logs backend
docker-compose logs frontend
```

---

## Running Locally (without Docker)

### Prerequisites
- Node.js 20+
- MongoDB running locally on port `27017`

### Backend

```bash
cd server
npm install
npm run dev
```

### Frontend

```bash
cd client
npm install
npm run dev
```

---

## Docker Architecture

```
Browser (host)
    │
    ├──► http://localhost:5173  ──► [frontend container]
    │                                     │ (Vite built bundle with
    │                                     │  VITE_API_URL baked in)
    │
    └──► http://localhost:5000  ──► [backend container]
                                          │
                                          └──► mongo:27017 ──► [mongo container]
```

> **Why `--build-arg` for `VITE_API_URL`?**  
> Vite statically replaces `import.meta.env.VITE_*` at **build time**, not runtime. The URL must be passed as a Docker build argument (`ARG`) so it is available during `npm run build` inside the image. Setting it via `docker run -e` or `env_file` at runtime is too late.

---

## CI/CD Pipeline

The GitHub Actions workflow (`.github/workflows/ci.yml`) runs on every **push** and **pull request** to `main`.

### `build-and-lint` job

| Step | Description |
|---|---|
| Checkout repo | Clone code into the runner |
| Set up Node.js 20 | Install correct Node version |
| Install backend deps | `npm ci` in `server/` |
| Lint backend | ESLint — fails on any warning |
| Install frontend deps | `npm ci` in `client/` |
| Lint frontend | ESLint — fails on any warning |
| Build backend image | Validates `server/Dockerfile` |
| Build frontend image | Validates `client/Dockerfile` + Vite build |

### `deploy` job

- Runs only on `push` to `main` (not on PRs)
- Requires `build-and-lint` to pass first
- SSHs into an EC2 instance and deploys the updated containers

---

## Data Persistence

MongoDB data is stored in a Docker named volume (`mongo-data`), so it persists across container restarts and `docker-compose down` calls. Data is only lost if the volume is explicitly removed:

```bash
docker-compose down -v   # WARNING: this deletes all MongoDB data
```

---

## AWS Deployment

The application is deployed on an **AWS EC2** instance. The CI/CD pipeline automatically deploys on every push to `main`.

### GitHub Secrets required

| Secret | Description |
|---|---|
| `EC2_HOST` | Public IP of the EC2 instance |
| `EC2_USER` | SSH user (e.g. `ubuntu`) |
| `EC2_SSH_KEY` | Private SSH key (PEM contents) to access the EC2 instance |

Go to **GitHub → Settings → Secrets and variables → Actions → New repository secret** to add these.

### How the deploy job works

1. Checks out the repo
2. Adds the EC2 IP to `known_hosts` (avoids SSH host verification prompts)
3. Writes the private key and sets correct permissions (`chmod 600`)
4. Copies `docker-compose.yml` and `server/.env` to the EC2 instance via `scp`
5. SSHs in and runs `docker-compose up -d --build` to restart with the latest code

### EC2 prerequisites

The EC2 instance must have Docker and Docker Compose installed. SSH in and run:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker ubuntu   # Allow running docker without sudo
```

---

## Kubernetes

Kubernetes manifests are in the `k8s/` directory.

### Apply all manifests

```bash
kubectl apply -f k8s/
```

### Components

| File | Creates |
|---|---|
| `mongo-deployment.yml` | MongoDB Deployment, headless Service, PersistentVolumeClaim |
| `backend-deployment.yml` | Express API Deployment, ClusterIP Service |
| `client-deployment.yml` | React frontend Deployment, LoadBalancer Service |

### Check status

```bash
kubectl get pods
kubectl get services
kubectl logs deploy/backend
kubectl logs deploy/client
```

> **Note on `VITE_API_URL` in K8s:** The frontend image has the API URL baked in at build time. To change it, rebuild the image with the correct `--build-arg VITE_API_URL=<url>` and redeploy.
