pipeline {
    agent any

    environment {
        DATABRICKS_CLI_VERSION = '0.236.0'
        PYTHON_VERSION = '3.10'
    }

    parameters {
        choice(
            name: 'DEPLOY_TARGET',
            choices: ['dev', 'prod'],
            description: 'Deployment target'
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

        stage('Setup') {
            steps {
                sh '''
                    # Install Databricks CLI
                    curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

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
                    sh 'pytest tests/'
                }
            }
        }

        stage('Validate Bundle') {
            steps {
                withCredentials([
                    string(credentialsId: "${params.DEPLOY_TARGET}-databricks-client-id", variable: 'DATABRICKS_CLIENT_ID'),
                    string(credentialsId: "${params.DEPLOY_TARGET}-databricks-client-secret", variable: 'DATABRICKS_CLIENT_SECRET')
                ]) {
                    dir('mlops_stack_demo') {
                        sh "databricks bundle validate -t ${params.DEPLOY_TARGET}"
                    }
                }
            }
        }

        stage('Integration Tests') {
            when {
                expression { params.RUN_INTEGRATION_TESTS == true }
            }
            steps {
                withCredentials([
                    string(credentialsId: "${params.DEPLOY_TARGET}-databricks-client-id", variable: 'DATABRICKS_CLIENT_ID'),
                    string(credentialsId: "${params.DEPLOY_TARGET}-databricks-client-secret", variable: 'DATABRICKS_CLIENT_SECRET')
                ]) {
                    dir('mlops_stack_demo') {
                        sh """
                            databricks bundle deploy -t ${params.DEPLOY_TARGET}
                            databricks bundle run write_feature_table_job -t ${params.DEPLOY_TARGET}
                            databricks bundle run model_training_job -t ${params.DEPLOY_TARGET}
                        """
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                allOf {
                    branch 'main'
                    expression { params.DEPLOY_TARGET == 'prod' }
                }
            }
            steps {
                withCredentials([
                    string(credentialsId: 'prod-databricks-client-id', variable: 'DATABRICKS_CLIENT_ID'),
                    string(credentialsId: 'prod-databricks-client-secret', variable: 'DATABRICKS_CLIENT_SECRET')
                ]) {
                    dir('mlops_stack_demo') {
                        sh 'databricks bundle deploy -t prod'
                    }
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
            echo 'Pipeline failed!'
        }
    }
}
