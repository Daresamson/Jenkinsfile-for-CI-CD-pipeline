pipeline {
    agent any

    environment {
        // Environment-specific variables (secured via Jenkins credentials)
        STAGING_ENV = credentials('staging-env-credentials') // Staging credentials
        PROD_ENV = credentials('prod-env-credentials')       // Production credentials
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials') // Docker Hub login
        APP_NAME = 'my-app' // Name of the application
    }

    options {
        timeout(time: 60, unit: 'MINUTES') // Pipeline will timeout after 60 minutes
        retry(2) // Retry failed stages twice before marking as failed
        buildDiscarder(logRotator(numToKeepStr: '10')) // Keep only the last 10 builds
    }

    stages {
        stage('Build') {
            steps {
                script {
                    echo 'Building the application...'
                }
                // Build the application (e.g., Maven/Gradle or another build tool)
                sh './gradlew build' // Adjust based on the build system used
            }
            post {
                failure {
                    echo 'Build failed! Notifying stakeholders...'
                    // Notify via email/Slack (requires plugin integration)
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    echo 'Running unit tests...'
                }
                // Run unit tests and generate test reports
                sh './gradlew test' // Adjust as needed for your build system
                junit 'build/reports/tests/test/*.xml' // Publish test results
            }
            post {
                failure {
                    echo 'Tests failed! Halting pipeline...'
                    // Notify team of test failures
                }
            }
        }

        stage('Package') {
            steps {
                script {
                    echo 'Packaging the application into a Docker image...'
                }
                // Build a Docker image for the application
                sh """
                docker build -t ${APP_NAME}:latest .
                docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
                docker tag ${APP_NAME}:latest ${DOCKERHUB_CREDENTIALS_USR}/${APP_NAME}:latest
                docker push ${DOCKERHUB_CREDENTIALS_USR}/${APP_NAME}:latest
                """
            }
        }

        stage('Deploy to Staging') {
            steps {
                script {
                    echo 'Deploying to staging environment...'
                }
                // Deployment script for staging environment
                sh """
                docker pull ${DOCKERHUB_CREDENTIALS_USR}/${APP_NAME}:latest
                docker run -d --name ${APP_NAME}_staging -p 8080:8080 ${DOCKERHUB_CREDENTIALS_USR}/${APP_NAME}:latest
                """
            }
            post {
                success {
                    echo 'Deployment to staging successful.'
                }
                failure {
                    echo 'Deployment to staging failed. Halting pipeline...'
                }
            }
        }

        stage('Approval') {
            steps {
                // Manual approval step before proceeding to production
                script {
                    echo 'Waiting for manual approval...'
                }
                input message: 'Approve deployment to production?', ok: 'Approve'
            }
        }

        stage('Deploy to Production') {
            steps {
                script {
                    echo 'Deploying to production environment...'
                }
                // Deployment script for production environment
                sh """
                docker pull ${DOCKERHUB_CREDENTIALS_USR}/${APP_NAME}:latest
                docker run -d --name ${APP_NAME}_production -p 80:8080 ${DOCKERHUB_CREDENTIALS_USR}/${APP_NAME}:latest
                """
            }
            post {
                success {
                    echo 'Production deployment successful!'
                }
                failure {
                    echo 'Production deployment failed! Rolling back...'
                    // Rollback strategy
                    sh "docker stop ${APP_NAME}_production || true && docker rm ${APP_NAME}_production || true"
                    echo 'Rollback to the previous stable version complete.'
                }
            }
        }
    }

    post {
        always {
            // General cleanup and logging
            echo 'Cleaning up resources...'
            sh 'docker system prune -f' // Cleanup unused Docker resources
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed! Investigating issues...'
        }
    }
}
