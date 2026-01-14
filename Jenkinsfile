pipeline {
    agent any

    environment {
        DATABRICKS_CLI_VERSION = '0.236.0'
        PYTHON_VERSION = '3.10'
        // Workspace URL - same for both staging and prod in this setup
        DATABRICKS_HOST = 'https://e2-demo-field-eng.cloud.databricks.com'
    }

    parameters {
        choice(
            name: 'DEPLOY_TARGET',
            choices: ['staging', 'prod'],
            description: 'Deployment target environment'
        )
        booleanParam(
            name: 'RUN_INTEGRATION_TESTS',
            defaultValue: false,
            description: 'Run integration tests on Databricks'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Environment') {
            steps {
                sh '''
                    # Install Databricks CLI
                    curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
                    databricks --version

                    # Install Python dependencies
                    cd mlops_stack_demo
                    pip install -r requirements.txt
                    pip install -r ../test-requirements.txt
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                dir('mlops_stack_demo') {
                    sh 'pytest tests/ -v'
                }
            }
        }

        stage('Validate Bundle') {
            environment {
                // Service Principal OAuth credentials from Jenkins credentials store
                DATABRICKS_CLIENT_ID = credentials("${params.DEPLOY_TARGET}-databricks-client-id")
                DATABRICKS_CLIENT_SECRET = credentials("${params.DEPLOY_TARGET}-databricks-client-secret")
            }
            steps {
                dir('mlops_stack_demo') {
                    sh "databricks bundle validate -t ${params.DEPLOY_TARGET}"
                }
            }
        }

        stage('Deploy Bundle') {
            when {
                anyOf {
                    expression { params.RUN_INTEGRATION_TESTS == true }
                    allOf {
                        branch 'main'
                        expression { params.DEPLOY_TARGET == 'prod' }
                    }
                }
            }
            environment {
                DATABRICKS_CLIENT_ID = credentials("${params.DEPLOY_TARGET}-databricks-client-id")
                DATABRICKS_CLIENT_SECRET = credentials("${params.DEPLOY_TARGET}-databricks-client-secret")
            }
            steps {
                dir('mlops_stack_demo') {
                    sh "databricks bundle deploy -t ${params.DEPLOY_TARGET}"
                }
            }
        }

        stage('Integration Tests') {
            when {
                expression { params.RUN_INTEGRATION_TESTS == true }
            }
            environment {
                DATABRICKS_CLIENT_ID = credentials("${params.DEPLOY_TARGET}-databricks-client-id")
                DATABRICKS_CLIENT_SECRET = credentials("${params.DEPLOY_TARGET}-databricks-client-secret")
            }
            steps {
                dir('mlops_stack_demo') {
                    sh """
                        echo "Running feature engineering job..."
                        databricks bundle run write_feature_table_job -t ${params.DEPLOY_TARGET}

                        echo "Running model training job..."
                        databricks bundle run model_training_job -t ${params.DEPLOY_TARGET}
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
        }
    }
}
