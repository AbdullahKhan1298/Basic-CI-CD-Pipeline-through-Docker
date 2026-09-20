# 🚀 Basic CI/CD Pipeline through Docker, Jenkins & Kubernetes

A hands-on implementation of a complete **Continuous Integration / Continuous Deployment** workflow — taking a Go web application from source code to a running, containerized service on **Google Kubernetes Engine (GKE)**, with automated testing, image builds, and environment-based deployments (dev → canary → production).

---

## 📌 Problem Statement

When I built this project, I wanted to answer a question that theory alone couldn't:

> *"How does code actually travel from a developer's commit to a live, running service — safely, repeatably, and without a human manually SSH-ing into a server?"*

Most tutorials explain CI/CD as a diagram of boxes and arrows. I wanted to **build the boxes myself** — write the application, containerize it, wire up the pipeline, and watch a single `git push` trigger tests, a Docker build, and a Kubernetes deployment end to end. This repo is the result of that exercise.

---

## 🎯 What This Project Does

This is a small **frontend/backend Go application** (`gceme`) that reports instance metadata (hostname, zone, project, IP, etc.), deployed through a fully automated pipeline:

1. A developer pushes code to a branch.
2. **Jenkins** picks up the change, spins up ephemeral build agents *inside Kubernetes itself*, and runs the test suite (`go test`).
3. If tests pass, Jenkins builds a **Docker image** of the application and pushes it to a container registry.
4. Jenkins then deploys that image to **GKE**, choosing the target environment based on the branch:
   - `master` → **Production**
   - `canary` → **Canary** (a controlled rollout alongside production)
   - any other branch → an isolated **Dev namespace**, spun up on demand

This mirrors how real engineering teams ship software: automated testing as a gate, immutable Docker images as the unit of deployment, and Kubernetes manifests as the source of truth for what's running where.

---

## 🧱 Architecture

```
 Developer          Jenkins (on K8s)              Container            Kubernetes Engine
 ─────────          ────────────────              Registry             ──────────────────
 git push   ──────▶  1. Checkout code
                      2. Run go test        ──────▶
                      3. Build Docker image ──────▶  Push image
                      4. Deploy via kubectl                     ──────▶  Dev / Canary / Prod
                                                                          namespace
```

- **Application layer:** Go HTTP service, runnable in `--frontend` or `--backend` mode, exposing `/`, `/version`, and `/healthz` endpoints.
- **Containerization:** A minimal `Dockerfile` builds the Go binary inside a `golang` base image.
- **CI/CD orchestration:** A declarative `Jenkinsfile` defines the pipeline as code — no manual configuration through the Jenkins UI.
- **Deployment target:** Kubernetes manifests under `k8s/` describe Services and Deployments for each environment (`dev`, `canary`, `production`).

---

## 🛠️ Tech Stack

| Layer                | Tool / Technology                     |
|-----------------------|----------------------------------------|
| Language              | Go                                     |
| Containerization      | Docker                                 |
| CI/CD Orchestration   | Jenkins (Declarative Pipeline, Kubernetes agents) |
| Container Orchestration | Kubernetes (Google Kubernetes Engine) |
| Image Registry        | Google Container Registry (GCR)        |
| Deployment Strategy   | Branch-based (dev / canary / production) |

---

## 📂 Repository Structure

```
.
├── Dockerfile          # Builds the Go app into a container image
├── Jenkinsfile         # Declarative pipeline: test → build → deploy
├── main.go             # Application entrypoint (frontend/backend modes)
├── html.go             # Frontend HTML template
├── main_test.go        # Unit tests run in the CI "Test" stage
├── Gopkg.toml/.lock     # Go dependency management (dep)
├── k8s/                # Kubernetes manifests per environment
│   ├── services/       # Shared Service definitions
│   ├── dev/            # Dev environment deployment
│   ├── canary/         # Canary rollout deployment
│   └── production/     # Production deployment
└── vendor/             # Vendored Go dependencies
```

---

## ⚙️ How the Pipeline Works

The `Jenkinsfile` defines the entire lifecycle as code:

- **Test stage** — runs inside a `golang` container spun up on demand by Jenkins' Kubernetes plugin, executing `go test` against the source.
- **Build & push stage** — uses Google Cloud Build to compile a Docker image tagged with the branch name and build number, then pushes it to GCR.
- **Deploy stages** — conditionally triggered based on the Git branch:
  - `canary` branch deploys to the canary environment for validation.
  - `master` branch deploys straight to production.
  - Any other branch gets its own throwaway namespace for isolated dev testing (with `ClusterIP` instead of a public load balancer, to avoid exposing dev builds externally).

This branch-to-environment mapping is the part I found most valuable to build myself — it's the same core idea behind trunk-based development and progressive delivery used in production engineering teams.

---

## 📚 What I Learned

Building this end-to-end (rather than just reading about it) taught me concepts I now apply directly in my QA/automation work:

- **Pipeline-as-code** — writing a `Jenkinsfile` instead of clicking through a UI, and why that matters for reproducibility and version control.
- **Ephemeral build agents** — how Jenkins can run its own build steps *inside* Kubernetes Pods rather than on a fixed build server.
- **Docker fundamentals** — image layering, working directories, and building a Go binary inside a container.
- **Environment isolation** — using Kubernetes namespaces to give every branch its own sandbox instead of one shared "staging" environment.
- **Deployment strategies** — the practical difference between a canary release and a full production rollout, and how manifests differ between them.
- **Testing as a pipeline gate** — how automated tests block a bad build from ever reaching a Docker image, let alone production.

This project directly shaped how I think about environments and release safety in my day-to-day QA work — testing isn't just "does the feature work," it's "does the *pipeline* stop a broken build before it ships."

---

## ▶️ Running It Locally

```bash
# Build the Docker image
docker build -t gceme:local .

# Run in backend mode
docker run -p 8081:8080 gceme:local --backend-service=http://localhost:8081

# Run in frontend mode (pointing at the backend above)
docker run -p 8080:8080 gceme:local --frontend --backend-service=http://localhost:8081
```

> Note: this app was originally built to read GCE instance metadata, so several fields (zone, project, internal/external IP) will only populate when actually running on Google Cloud infrastructure.

---

## 🔭 Future Improvements

If I revisited this project today, I'd extend it with:
- GitHub Actions as an alternative to Jenkins, to compare pipeline-as-code approaches
- Automated rollback on failed health checks
- Integration tests hitting the deployed environment post-deploy, not just unit tests pre-build

---

## 🙏 Acknowledgment

This project was built as a hands-on lab exercise based on Google Cloud's reference architecture for CI/CD on Kubernetes Engine, adapted here for my own learning and experimentation with Docker, Jenkins, and GKE.
