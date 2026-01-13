# MLOps Setup Guide
[(back to main README)](../README.md)

## Table of contents
* [Intro](#intro)
* [Create a hosted Git repo](#create-a-hosted-git-repo)
* [Configure CI/CD](#configure-cicd---github-actions)
* [Merge PR with initial ML code](#merge-a-pr-with-your-initial-ml-code)
* [Create release branch](#create-release-branch)

* [Deploy ML resources and enable production jobs](#deploy-ml-resources-and-enable-production-jobs)
* [Next steps](#next-steps)

## Intro
This page explains how to productionize the current project, setting up CI/CD and
ML resource deployment, and deploying ML training and inference jobs.

After following this guide, data scientists can follow the [ML Pull Request](ml-pull-request.md) guide to make changes to ML code or deployed jobs.

## Create a hosted Git repo
Create a hosted Git repo to store project code, if you haven't already done so. From within the project
directory, initialize Git and add your hosted Git repo as a remote:
```
git init --initial-branch=main
```

```
git remote add upstream <hosted-git-repo-url>
```

Commit the current `README.md` file and other docs to the `main` branch of the repo, to enable forking the repo:
```

git add README.md docs .gitignore mlops_stack_demo/resources/README.md
git commit -m "Adding project README"

git push upstream main
```

## Configure CI/CD - GitHub Actions

### Prerequisites
* You must be an account admin to add service principals to the account.
* You must be a Databricks workspace admin in the staging and prod workspaces. 
  Verify that you're an admin by viewing the
  [staging workspace admin console](https://e2-dogfood.staging.cloud.databricks.com#setting/accounts) and
  [prod workspace admin console](https://e2-demo-field-eng.cloud.databricks.com#setting/accounts). 
  If the admin console UI loads instead of the Databricks workspace homepage, you are an admin.

### Set up authentication for CI/CD
#### Set up Service Principal

To authenticate and manage ML resources created by CI/CD, 
[service principals](https://docs.databricks.com/administration-guide/users-groups/service-principals.html)
for the project should be created and added to both staging and prod workspaces. Follow
[Add a service principal to your Databricks account](https://docs.databricks.com/administration-guide/users-groups/service-principals.html#add-a-service-principal-to-your-databricks-account)
and [Add a service principal to a workspace](https://docs.databricks.com/administration-guide/users-groups/service-principals.html#add-a-service-principal-to-a-workspace)
for details.


For your convenience, we also have a [Terraform module](https://registry.terraform.io/modules/databricks/mlops-aws-project/databricks/latest) that can set up your service principals.



#### Configure Service Principal (SP) permissions 
If the created project uses **Unity Catalog**, we expect a catalog to exist with the name of the deployment target by default. 
For example, if the deployment target is dev, we expect a catalog named dev to exist in the workspace. 
If you want to use different catalog names, please update the target names declared in the[mlops_stack_demo/databricks.yml](../mlops_stack_demo/databricks.yml) file.
If changing the staging, prod, or test deployment targets, you'll also need to update the workflows located in the .github/workflows directory.

The SP must have proper permission in each respective environment and the catalog for the environments.

For the integration test and the ML training job, the SP must have permissions to read the input Delta table and create experiment and models. 
i.e. for each environment:
- USE_CATALOG
- USE_SCHEMA
- MODIFY
- CREATE_MODEL
- CREATE_TABLE

For the batch inference job, the SP must have permissions to read input Delta table and modify the output Delta table. 
i.e. for each environment
- USAGE permissions for the catalog and schema of the input and output table.
- SELECT permission for the input table.
- MODIFY permission for the output table if it pre-dates your job.


#### Set secrets for CI/CD

After creating the service principals and adding them to the respective staging and prod workspaces, you need to generate access tokens and store them as GitHub secrets.

##### Step 1: Generate OAuth Secret for the Service Principal

1. Go to Databricks Account Console → Service Principals → Select your SP
2. Open the **Secrets** tab
3. Under **OAuth secrets**, click "Generate secret"
4. Set the secret's lifetime (up to 2 years)
5. **Copy the displayed Client ID and Secret** - the secret is shown only once!

##### Step 2: Generate Access Token Using OAuth

Use the service principal's Client ID and OAuth Secret to request an access token:

1. Construct the token endpoint URL for your workspace:
   ```
   https://<databricks-instance>/oidc/v1/token
   ```

   For example: `https://e2-demo-field-eng.cloud.databricks.com/oidc/v1/token`

2. Use `curl` to obtain the access token:
   ```bash
   curl --request POST \
     --url "https://<databricks-instance>/oidc/v1/token" \
     --header "Content-Type: application/x-www-form-urlencoded" \
     --data "client_id=<service-principal-client-id>" \
     --data "client_secret=<service-principal-secret>" \
     --data "grant_type=client_credentials" \
     --data "scope=all-apis"
   ```

3. Copy the `access_token` from the JSON response.

##### Step 3: Store Access Tokens as GitHub Secrets

In your GitHub repository, go to **Settings** → **Secrets and variables** → **Actions** and add:

* `STAGING_WORKSPACE_TOKEN` : Access token for staging workspace service principal
* `PROD_WORKSPACE_TOKEN` : Access token for prod workspace service principal

##### Step 4: Reference Tokens in GitHub Actions

Your workflow will use these environment variables:
```yaml
env:
  DATABRICKS_HOST: https://your-workspace.cloud.databricks.com
  DATABRICKS_TOKEN: ${{ secrets.STAGING_WORKSPACE_TOKEN }}
```

##### Alternative: Workload Identity Federation (OIDC)

For stronger security without managing static secrets, you can configure Databricks to accept OIDC federated tokens directly from GitHub Actions:

1. Set up a federation policy linking your Databricks service principal to your GitHub repository
2. Configure your workflow:
   ```yaml
   permissions:
     id-token: write
     contents: read
   env:
     DATABRICKS_AUTH_TYPE: github-oidc
     DATABRICKS_HOST: <your-databricks-workspace-url>
     DATABRICKS_CLIENT_ID: <service-principal-client-id>
   ```

This removes the need to manually manage and rotate secrets.

##### Additional Secrets

* `WORKFLOW_TOKEN` : [Github token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic) with workflow permissions. This secret is needed for the Deploy CI/CD Workflow.

##### Best Practices

* **Never** store access tokens in code or plaintext—always use GitHub Actions secrets
* Rotate tokens regularly and monitor expiry
* Use dedicated service principals and separate tokens per environment (staging, prod)

Next, be sure to update the [Workflow Permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#modifying-the-permissions-for-the-github_token) section under Repo Settings > Actions > General:
- Allow `Read and write permissions`,
- Allow workflows to be able to open pull requests (PRs).


### Setting up CI/CD workflows
After setting up authentication for CI/CD, you can now set up CI/CD workflows. We provide a [Deploy CICD workflow](../.github/workflows/deploy-cicd.yml) that can be used to generate the other CICD workflows mentioned below for projects. 
This workflow is manually triggered with `project_name` as parameter. This workflow will need to be triggered for each project to set up its set of CI/CD workflows that can be used to deploy ML resources and run ML jobs in the staging and prod workspaces. 
These workflows will be defined under `.github/workflows`.

If you want to deploy CI/CD for an initialized project (`Project-Only` MLOps Stacks initialization), you can manually run the `deploy-cicd.yml` workflow from the [Github Actions UI](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow?tool=webui) once the project code has been added to your main repo. 
The workflow will create a pull request with all the changes against your main branch. Review and approve it to commit the files to deploy CI/CD for the project. 



## Merge a PR with your initial ML code
Create and push a PR branch adding the ML code to the repository.

```
git checkout -b add-ml-code
git add .
git commit -m "Add ML Code"
git push upstream add-ml-code
```

Open a PR from the newly pushed branch. CI will run to ensure that tests pass
on your initial ML code. Fix tests if needed, then get your PR reviewed and merged.
After the pull request merges, pull the changes back into your local `main`
branch:

```
git checkout main
git pull upstream main
```

## Create release branch
Create and push a release branch called `release` off of the `main` branch of the repository:
```
git checkout -b release main
git push upstream release
git checkout main
```

Your production jobs (model training, batch inference) will pull ML code against this branch, while your staging jobs will pull ML code against the `main` branch. Note that the `main` branch will be the source of truth for ML resource configs and CI/CD workflows.

For future ML code changes, iterate against the `main` branch and regularly deploy your ML code from staging to production by merging code changes from the `main` branch into the `release` branch.

## Deploy ML resources and enable production jobs
Follow the instructions in [mlops_stack_demo/resources/README.md](../mlops_stack_demo/resources/README.md) to deploy ML resources
and production jobs.

## Next steps
After you configure CI/CD and deploy training & inference pipelines, notify data scientists working
on the current project. They should now be able to follow the
[ML pull request guide](ml-pull-request.md) and 
[ML resource config guide](../mlops_stack_demo/resources/README.md)  to propose, test, and deploy
ML code and pipeline changes to production.
