# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Databricks MLOps Stacks project for automated ML pipeline management. It implements a production-grade pipeline for training, validating, and deploying a regression model (NYC taxi fare prediction example).

## Commands

### Running Tests
```bash
cd mlops_stack_demo
pip install -r requirements.txt
pip install -r ../test-requirements.txt
pytest tests                    # Run all tests
pytest tests/feature_engineering/pickup_features_test.py  # Single test file
```
Note: Tests require Java 11+ for local Spark session.

### Databricks Bundle Commands
All bundle commands must be run from the `mlops_stack_demo/` directory:
```bash
databricks bundle validate -t dev     # Validate config
databricks bundle deploy -t dev       # Deploy resources to dev workspace
databricks bundle run <job_name> -t dev  # Run a specific job
databricks bundle destroy -t dev      # Remove deployed resources
```

Deployment targets: `dev`, `prod`

## Architecture

### ML Pipeline Flow
1. **Feature Engineering** (`feature_engineering/`) - Computes features and writes to Feature Store
2. **Training** (`training/`) - Trains model using Feature Store data
3. **Validation** (`validation/`) - Validates model quality before deployment
4. **Deployment** (`deployment/model_deployment/`) - Assigns model alias for production
5. **Batch Inference** (`deployment/batch_inference/`) - Runs scheduled predictions
6. **Monitoring** (`monitoring/`) - Tracks model/feature drift, triggers retraining

### Bundle Configuration
- `mlops_stack_demo/databricks.yml` - Root bundle config defining targets and variables
- `mlops_stack_demo/resources/*.yml` - Job/workflow definitions for each pipeline component

### Environment-Target Mapping
| Target | Catalog | Purpose |
|--------|---------|---------|
| dev | dev | Local development & CI integration tests |
| prod | fins_genai | Production (auto-deployed from main branch) |

### Feature Modules
Feature computation logic lives in `feature_engineering/features/`. Each module implements `compute_features_fn` that transforms input DataFrames into feature columns for the Feature Store.

## CI/CD

- PRs trigger unit tests + integration tests against dev workspace
- Merge to main deploys to prod workspace
