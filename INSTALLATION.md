# Installation & Setup

This document describes how to run the **Cyber-Security Agent** locally, in Docker, and in the cloud (Azure Container Apps and Google Cloud Run).

> If you are following the *AI in Production* course, this is a compact reference for all the steps you need for this project.

---

## 1. Prerequisites

- **Python** 3.12+
- **Node.js** 20+ and npm
- **uv** (Python package manager) – https://astral.sh/uv
- **Docker**
- **Terraform** 1.0+
- **Azure CLI** (for Azure deployments)
- **gcloud CLI** (for GCP deployments)
- **Semgrep account** at https://semgrep.dev to obtain `SEMGREP_APP_TOKEN`

---

## 2. Environment Configuration

Create a `.env` file in the **project root** (`cyber-security-agent/`):

```bash
OPENROUTER_API_KEY=your-openrouter-api-key
OPENAI_API_KEY=your-openai-api-key-optional
SEMGREP_APP_TOKEN=your-semgrep-app-token
```

Notes:

- The backend requires **`OPENROUTER_API_KEY`** to run analyses.
- `SEMGREP_APP_TOKEN` must be a valid Semgrep App token with Web API / Agent scopes.
- `.env` is already listed in `.gitignore`; do **not** commit it.

---

## 3. Run Locally (without Docker)

### 3.1 Backend (FastAPI + Agents)

From the repo root:

```bash
cd backend

# Ensure uv is installed (see https://astral.sh/uv)
uv --version  # should print a version

# Run the FastAPI server (uses pyproject.toml and uv.lock)
uv run server.py
```

You should see Uvicorn logs similar to:

```text
INFO:     Uvicorn running on http://127.0.0.1:8000
```

The backend will:

- Load `OPENROUTER_API_KEY`, `OPENAI_API_KEY`, and `SEMGREP_APP_TOKEN` from the root `.env`
- Expose `POST /api/analyze` and `GET /health` on `http://localhost:8000`

### 3.2 Frontend (Next.js)

In a **second** terminal:

```bash
cd frontend
npm install
npm run dev
```

Open:

- http://localhost:3000

The frontend will automatically call the backend at `http://localhost:8000/api/analyze` in development (no extra env vars needed).

### 3.3 Test the Analyzer

1. Visit `http://localhost:3000`
2. Click **“Choose File”** and select `airline.py` from the repo root
3. Click **“Analyze Code”**
4. You should see a summary and a list of detected vulnerabilities

---

## 4. Run with Docker

You can run the entire stack in a single container, matching the cloud deployment.

From the repo root:

```bash
# Build the image (first time will take a few minutes)
docker build -t cyber-analyzer .

# Run the container, injecting your .env
docker run --rm \
  --name cyber-analyzer \
  -p 8000:8000 \
  --env-file .env \
  cyber-analyzer
```

Then open:

- http://localhost:8000

The container:

- Serves the FastAPI API on `/api/*`
- Serves the built Next.js frontend on `/`

Stop with `Ctrl+C` to terminate the container.

---

## 5. Deploy to Azure (Container Apps + Terraform)

This section summarizes the Azure setup and deployment steps for this project.

### 5.1 One-time Azure Setup

1. Create an Azure account and subscription
2. Create a resource group (default used by Terraform is `cyber-analyzer-rg`)
3. Install and log in with Azure CLI:

```bash
az login
```

4. Register required resource providers (only once per subscription):

```bash
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

Wait until both report `Registered`.

### 5.2 Export Environment Variables

From the repo root (Mac/Linux example):

```bash
# Load keys from .env
export $(cat .env | xargs)

# Optional: echo a short prefix to confirm
echo "OpenRouter key: ${OPENROUTER_API_KEY:0:8}..."
echo "Semgrep token:  ${SEMGREP_APP_TOKEN:0:8}..."
```

On Windows PowerShell, you can load variables from `.env` with:

```powershell
Get-Content .env | ForEach-Object {
  $name, $value = $_.Split('=', 2)
  Set-Item -Path "env:$name" -Value $value
}
```

### 5.3 Initialize Terraform for Azure

```bash
cd terraform/azure

terraform init

# Use an 'azure' workspace to isolate state
terraform workspace new azure || terraform workspace select azure
terraform workspace show
```

### 5.4 Plan & Apply

```bash
terraform plan \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -var="openrouter_api_key=$OPENROUTER_API_KEY" \
  -var="semgrep_app_token=$SEMGREP_APP_TOKEN"

terraform apply \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -var="openrouter_api_key=$OPENROUTER_API_KEY" \
  -var="semgrep_app_token=$SEMGREP_APP_TOKEN"
```

Terraform will:

- Build the Docker image from the repo root
- Push it to Azure Container Registry (ACR)
- Create a Container Apps environment
- Deploy the container with your environment variables

When finished, get the URL:

```bash
terraform output app_url
```

Open the printed HTTPS URL in your browser and test the analyzer.

### 5.5 Clean Up (Azure)

To avoid charges when you are done:

```bash
cd terraform/azure

terraform destroy \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -var="openrouter_api_key=$OPENROUTER_API_KEY" \
  -var="semgrep_app_token=$SEMGREP_APP_TOKEN"
```

You can optionally delete the resource group afterwards using the Azure Portal or `az group delete`.

---

## 6. Deploy to Google Cloud Run (Terraform)

This section summarizes the Google Cloud Platform setup and Cloud Run deployment steps for this project.
The production deployment behind **https://www.codesecurity.pp.ua/** follows this pattern.

### 6.1 One-time GCP Setup

1. Create a GCP account and project (e.g. `cyber-analyzer`)
2. Link billing and set budget alerts
3. Install and initialize `gcloud`:

```bash
gcloud init
```

4. Note your **Project ID** (e.g., `cyber-analyzer-123456`).

5. Enable required APIs:

```bash
gcloud services enable run.googleapis.com containerregistry.googleapis.com cloudbuild.googleapis.com
```

### 6.2 Export Environment Variables

From the repo root (Mac/Linux example):

```bash
# Load keys from .env
export $(cat .env | xargs)

# Set the project ID used by Terraform
export TF_VAR_project_id="your-gcp-project-id"
```

On Windows PowerShell, set `$env:TF_VAR_project_id` accordingly.

### 6.3 Authenticate for Terraform

```bash
gcloud auth login
gcloud config set project "$TF_VAR_project_id"

# Application default credentials used by the Google provider
gcloud auth application-default login

# Optional: align quota project
gcloud auth application-default set-quota-project "$TF_VAR_project_id"

# Configure Docker for Artifact Registry
gcloud auth configure-docker
```

### 6.4 Initialize Terraform for GCP

```bash
cd terraform/gcp

terraform init

terraform workspace new gcp || terraform workspace select gcp
terraform workspace show
```

### 6.5 Plan & Apply

```bash
terraform plan \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -var="openrouter_api_key=$OPENROUTER_API_KEY" \
  -var="semgrep_app_token=$SEMGREP_APP_TOKEN"

terraform apply \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -var="openrouter_api_key=$OPENROUTER_API_KEY" \
  -var="semgrep_app_token=$SEMGREP_APP_TOKEN"
```

Terraform will:

- Enable required GCP APIs if needed
- Build the Docker image and push it to Artifact Registry
- Create a Cloud Run service with public ingress
- Inject your environment variables

After apply completes, get the service URL:

```bash
terraform output service_url
```

Open the URL in your browser to use the analyzer. You can then map a custom domain (such as `https://www.codesecurity.pp.ua/`) via the Cloud Run console.

### 6.6 Clean Up (GCP)

To avoid charges when you are done:

```bash
cd terraform/gcp

terraform destroy \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -var="openrouter_api_key=$OPENROUTER_API_KEY" \
  -var="semgrep_app_token=$SEMGREP_APP_TOKEN"
```

You can also delete any remaining container images from Artifact Registry if needed.

---

## 7. Troubleshooting

For more detailed troubleshooting, refer to the official documentation for:
- Azure Container Apps, Azure Container Registry, and Terraform
- Google Cloud Run, Artifact Registry, and Terraform

Common tips:

- Ensure `.env` is in the **project root** and has no extra spaces around `=`.
- Verify `OPENROUTER_API_KEY` and `SEMGREP_APP_TOKEN` are set before starting the backend.
- For Cloud Run and Container Apps, check logs in the respective cloud consoles if the UI responds with errors.
