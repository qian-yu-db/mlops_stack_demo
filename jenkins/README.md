# Local Jenkins Setup

This guide follows the [Microsoft Azure Databricks Jenkins documentation](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/ci-cd/jenkins).

## Quick Start

### 1. Start Jenkins
```bash
cd jenkins
docker-compose up -d
```

### 2. Get Initial Admin Password
```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### 3. Access Jenkins
Open http://localhost:8080 and paste the admin password.

### 4. Install Suggested Plugins
Select "Install suggested plugins" during setup.

### 5. Create Admin User
Create your admin account when prompted.

---

## Configure Jenkins for This Project

### Install Additional Plugins
Go to: Manage Jenkins → Plugins → Available plugins

Install:
- **Pipeline**
- **Git**
- **Credentials Binding Plugin**

### Add Databricks Service Principal Credentials

The Databricks CLI uses OAuth M2M (machine-to-machine) authentication with service principals. You need to add **Client ID** and **Client Secret** for each environment.

Go to: **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

#### Staging Environment Credentials

| Field | Value |
|-------|-------|
| Kind | Secret text |
| Scope | Global |
| Secret | Your Service Principal Client ID |
| ID | `staging-databricks-client-id` |
| Description | Staging workspace SP Client ID |

| Field | Value |
|-------|-------|
| Kind | Secret text |
| Scope | Global |
| Secret | Your Service Principal Client Secret |
| ID | `staging-databricks-client-secret` |
| Description | Staging workspace SP Client Secret |

#### Production Environment Credentials

| Field | Value |
|-------|-------|
| Kind | Secret text |
| Scope | Global |
| Secret | Your Service Principal Client ID |
| ID | `prod-databricks-client-id` |
| Description | Prod workspace SP Client ID |

| Field | Value |
|-------|-------|
| Kind | Secret text |
| Scope | Global |
| Secret | Your Service Principal Client Secret |
| ID | `prod-databricks-client-secret` |
| Description | Prod workspace SP Client Secret |

### How to Get Service Principal Credentials

1. Go to **Databricks Account Console** → **Service Principals**
2. Select or create a service principal
3. Open the **Secrets** tab
4. Under **OAuth secrets**, click **Generate secret**
5. Copy the **Client ID** and **Secret** (secret shown only once!)

See [docs/mlops-setup.md](../docs/mlops-setup.md) for detailed instructions.

### Create Pipeline Job

1. **New Item** → Enter name "mlops-stack-demo" → **Pipeline**
2. In **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: Your GitHub repo URL or local path
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
3. Click **Save**

---

## How the Pipeline Works

The Jenkinsfile uses environment variables that the Databricks CLI automatically picks up:

| Variable | Description |
|----------|-------------|
| `DATABRICKS_HOST` | Workspace URL (set globally in Jenkinsfile) |
| `DATABRICKS_CLIENT_ID` | Service Principal Client ID (from Jenkins credentials) |
| `DATABRICKS_CLIENT_SECRET` | Service Principal Secret (from Jenkins credentials) |

The `credentials()` function in Jenkins binds secrets to environment variables:
```groovy
environment {
    DATABRICKS_CLIENT_ID = credentials("${params.DEPLOY_TARGET}-databricks-client-id")
    DATABRICKS_CLIENT_SECRET = credentials("${params.DEPLOY_TARGET}-databricks-client-secret")
}
```

---

## GitHub Integration Options

### Option A: Polling (No webhook needed)
In your Pipeline job configuration:
- Build Triggers → Poll SCM
- Schedule: `H/5 * * * *` (every 5 minutes)

### Option B: Webhooks with ngrok (for real-time triggers)
```bash
# Install ngrok
brew install ngrok

# Expose Jenkins to internet
ngrok http 8080
```
Use the ngrok URL as your GitHub webhook:
`https://xxxx.ngrok.io/github-webhook/`

---

## Install Dependencies in Jenkins Container

After Jenkins is running, install required tools:

```bash
# Enter Jenkins container as root
docker exec -it -u root jenkins bash

# Install curl, Python, and pip
apt-get update && apt-get install -y curl python3 python3-pip python3-venv

# Install Databricks CLI
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

# Verify installation
databricks --version

# Exit container
exit
```

---

## Test the Pipeline

1. Go to your pipeline job
2. Click **Build with Parameters**
3. Select **DEPLOY_TARGET**: `staging` or `prod`
4. Optionally check **RUN_INTEGRATION_TESTS** (requires valid credentials)
5. Click **Build**

### Pipeline Stages

| Stage | Description |
|-------|-------------|
| Checkout | Clones the repository |
| Setup Environment | Installs Databricks CLI and Python dependencies |
| Unit Tests | Runs pytest locally |
| Validate Bundle | Validates DAB configuration against Databricks |
| Deploy Bundle | Deploys resources to Databricks (conditional) |
| Integration Tests | Runs feature engineering and model training jobs |

---

## Troubleshooting

### Cannot authenticate to Databricks
- Verify Client ID and Secret are correct in Jenkins credentials
- Ensure service principal is added to the target workspace
- Check that service principal has proper permissions

### Connection issues
```bash
docker exec jenkins curl -I https://e2-demo-field-eng.cloud.databricks.com
```

### Pipeline fails at pytest
Python should be installed during setup, but if issues persist:
```bash
docker exec -it -u root jenkins bash
apt-get update && apt-get install -y python3 python3-pip openjdk-11-jdk
```
Note: Tests require Java 11+ for local Spark session.

### View Databricks CLI auth status
```bash
docker exec jenkins sh -c 'DATABRICKS_HOST=https://e2-demo-field-eng.cloud.databricks.com DATABRICKS_CLIENT_ID=<id> DATABRICKS_CLIENT_SECRET=<secret> databricks auth describe'
```

---

## Stop and Reset Jenkins

### Stop Jenkins
```bash
docker-compose down
```

### Reset Jenkins (fresh start)
```bash
docker-compose down -v  # Removes volumes too
docker-compose up -d
```
