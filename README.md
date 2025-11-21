# Cyber-Security Agent

[![Live Website](https://img.shields.io/badge/Live_Website-6c63ff?logo=rocket&logoColor=white&labelColor=5a52d3)](https://www.codesecurity.pp.ua/)

An **LLM-first Cybersecurity Analyzer** that inspects Python code for security vulnerabilities. Rather than being just a Semgrep UI, it showcases how to build a **reliable, tool-using LLM agent** for security-critical workloads. It combines:

- **OpenAI-compatible LLMs** (via OpenRouter/LiteLLM and `openai-agents`) for structured reasoning and tool-calling
- **Semgrep** for static security scanning, invoked as a tool by the LLM
- A modern **Next.js** frontend
- A **FastAPI** backend
- Containerized deployment to **Google Cloud Run** (and optionally **Azure Container Apps**) using **Terraform**

- **Live Demo**: https://www.codesecurity.pp.ua/
- **Backend**: FastAPI running in a single container on Cloud Run/Azure
- **Frontend**: Next.js 15 UI served by FastAPI as static files
- **Security Engine**:
  - LLM agent orchestrated with `openai-agents`
  - Semgrep MCP server (`semgrep-mcp`) for deep static analysis

---

## Features

- **Upload Python files** (`.py`) and run a full security analysis
- **LLM-first analysis pipeline**:
  - The LLM agent always calls Semgrep once, then performs its own reasoning on top of the findings.
  - Produces a single, consolidated security report you can trust.
- **Hybrid scanning**:
  - Semgrep static analysis via MCP
  - LLM-powered reasoning that adds missing findings, context, and prioritized fixes
- **Structured, typed security reports**:
  - Executive summary
  - Per-issue title, description, vulnerable snippet, recommended fix
  - CVSS score and severity (critical / high / medium / low), validated via Pydantic models
- **Production-ready deployment**:
  - Single Docker image
  - Terraform modules for:
    - Azure Container Apps
    - Google Cloud Run (used in production at https://www.codesecurity.pp.ua/)
- **Cloud-friendly defaults**:
  - Scales to zero on Cloud Run and Azure
  - 1 vCPU / 2GiB RAM tuned for Semgrep and the LLM tooling

---

## Architecture Overview

- **Frontend** (`frontend/`)
  - Next.js 15 (App Router), React 19, TypeScript
  - Main page: `src/app/page.tsx`
    - Handles file upload, calls backend `/api/analyze`, renders results
  - Components:
    - `CodeInput` – file picker + code viewer
    - `AnalysisResults` – summary & issues table with severity badges

- **Backend** (`backend/`)
  - FastAPI app in `server.py`
  - Key endpoints:
    - `POST /api/analyze` — analyze Python code and return a `SecurityReport`
    - `GET /health` — basic health check
    - `GET /network-test` — checks Semgrep API reachability
    - `GET /semgrep-test` — verifies Semgrep CLI can be installed and run
  - Loads `.env` with `python-dotenv`
  - Mounts built frontend under `/` from the `static/` folder in production

- **Agents & Semgrep MCP** (`backend/context.py`, `backend/mcp_servers.py`)
  - Security agent configured with `SECURITY_RESEARCHER_INSTRUCTIONS`
  - Uses `openai-agents` + LiteLLM to drive a tool-using LLM that talks to `semgrep-mcp` over MCP
  - Enforces:
    - Single Semgrep scan per request
    - `config: "auto"` for all scans
  - LLM outputs are parsed into the `SecurityReport` Pydantic model, ensuring stable, strongly typed responses

- **Infrastructure & Deployment** (`terraform/`)
  - **Dockerfile** at repo root:
    - Builds frontend (`npm ci && npm run build`)
    - Installs backend with `uv` from `pyproject.toml` / `uv.lock`
    - Serves app via `uv run uvicorn server:app --host 0.0.0.0 --port 8000`
  - **Azure** (`terraform/azure`)
    - Azure Container Registry (ACR)
    - Azure Container Apps environment
    - Container App with public ingress on port 8000
  - **GCP** (`terraform/gcp`)
    - Artifact Registry
    - Cloud Run service with public URL
    - 1 vCPU / 2GiB RAM, minScale 0, maxScale 1

---

## API Endpoints

- **POST `/api/analyze`**
  - Request body:
    - `{ "code": "Python source as a string" }`
  - Response (simplified):
    - `summary: string`
    - `issues: Array<{ title, description, code, fix, cvss_score, severity }>`
- **GET `/health`**
  - Returns `{ "message": "Cybersecurity Analyzer API" }`
- **GET `/network-test`**
  - Connectivity check to `https://semgrep.dev/api/v1/`
- **GET `/semgrep-test`**
  - Verifies Semgrep installation and version inside the running container

---

## Quick Start (Local Development)

For full, step-by-step setup (including Docker, Azure, and GCP), see **[INSTALLATION.md](./INSTALLATION.md)**.  
Below is a minimal local workflow.

1. **Clone the repo**

   ```bash
   git clone <this-repo-url>
   cd cyber-security-agent
   ```

2. **Create `.env` in the project root**

   ```bash
   # .env (do NOT commit this file)
   OPENROUTER_API_KEY=your-openrouter-api-key
   OPENAI_API_KEY=your-openai-api-key-optional
   SEMGREP_APP_TOKEN=your-semgrep-app-token
   ```

3. **Start the backend (FastAPI)**

   ```bash
   cd backend
   uv run server.py
   # Backend will listen on http://localhost:8000
   ```

4. **Start the frontend (Next.js)**

   ```bash
   cd frontend
   npm install
   npm run dev
   # Frontend at http://localhost:3000
   ```

5. **Use the app**

   - Open `http://localhost:3000`
   - Click **“Choose File”** and select a Python file (e.g. `airline.py` in the repo root)
   - Click **“Analyze Code”** to see security findings

---

## Environment Variables

All secrets are loaded from a `.env` file in the **project root**:

- `OPENROUTER_API_KEY` — required; used by the LLM agent
- `OPENAI_API_KEY` — optional; for OpenAI-compatible backends
- `SEMGREP_APP_TOKEN` — required Semgrep token for the MCP server
- `ENVIRONMENT` — set to `production` in cloud deployments
- `PYTHONUNBUFFERED` — set to `1` in containers for unbuffered logs

> **Security note:**  
> Never commit `.env` to version control. Keep your keys private and rotate them if they are ever exposed.

---

## Deployment

The production instance of this project is deployed to **Google Cloud Run** and available at:

- **https://www.codesecurity.pp.ua/**

This repo includes Terraform configurations for both Azure and GCP:

- **Azure Container Apps** — `terraform/azure`
- **Google Cloud Run** — `terraform/gcp` (used for `codesecurity.pp.ua`)

High-level workflow:

1. Build and push the Docker image using Terraform (Azure or GCP module)
2. Provision the container runtime (Container Apps / Cloud Run)
3. Inject environment variables (`OPENROUTER_API_KEY`, `OPENAI_API_KEY`, `SEMGREP_APP_TOKEN`)
4. Point your domain (optional) at the generated app URL

See **[INSTALLATION.md](./INSTALLATION.md)** for full setup and deployment commands.

---

## Tech Stack

| Category            | Technologies |
|---------------------|-------------|
| **Frontend**        | Next.js 15, React 19, TypeScript, Tailwind-style utility classes |
| **Backend**         | FastAPI, Uvicorn, `openai-agents`, LiteLLM, MCP |
| **Security**        | Semgrep MCP server (`semgrep-mcp`), CVSS scoring |
| **Infrastructure**  | Docker, Terraform, Google Cloud Run, Azure Container Apps |
| **Tooling**         | `uv` for Python env management, Node.js/npm for frontend |

---

## Course Guides

This project is used in the **AI in Production** course.
All required setup and deployment steps are documented in this README and in **[INSTALLATION.md](./INSTALLATION.md)**.

---

## License

This project is licensed under the MIT License — see the [LICENSE](./LICENSE) file for details.