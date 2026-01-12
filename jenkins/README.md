# Local Jenkins Setup

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
- **Credentials Binding**

### Add Databricks Credentials
Go to: Manage Jenkins → Credentials → System → Global credentials → Add Credentials

| Field | Value |
|-------|-------|
| Kind | Secret text |
| Secret | Your Databricks PAT |
| ID | `dev-databricks-token` |
| Description | Dev workspace token |

Repeat for `prod-databricks-token`.

### Create Pipeline Job
1. New Item → Enter name "mlops-stack-demo" → Pipeline
2. Pipeline section:
   - Definition: "Pipeline script from SCM"
   - SCM: Git
   - Repository URL: `/path/to/mlops_stack_demo` (local) or GitHub URL
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`

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

## Install Databricks CLI in Jenkins

After Jenkins is running, install Databricks CLI:

```bash
# Enter Jenkins container
docker exec -it -u root jenkins bash

# Install curl and Databricks CLI
apt-get update && apt-get install -y curl
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

# Verify installation
databricks --version

# Exit container
exit
```

---

## Test the Pipeline

1. Go to your pipeline job
2. Click "Build with Parameters"
3. Select DEPLOY_TARGET: `dev`
4. Check RUN_INTEGRATION_TESTS if needed
5. Click "Build"

---

## Troubleshooting

### Cannot connect to Databricks
- Verify token is correct in credentials
- Check network connectivity from container:
  ```bash
  docker exec jenkins curl -I https://your-workspace.cloud.databricks.com
  ```

### Pipeline fails at pytest
- Install Python in Jenkins container:
  ```bash
  docker exec -it -u root jenkins bash
  apt-get update && apt-get install -y python3 python3-pip
  ```

### Stop Jenkins
```bash
docker-compose down
```

### Reset Jenkins (fresh start)
```bash
docker-compose down -v  # Removes volumes too
docker-compose up -d
```
