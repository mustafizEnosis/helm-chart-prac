pipeline {
    agent any // Or a specific agent with kubectl/helm installed

    environment {
        KUBERNETES_NAMESPACE = 'argocd'           // The Kubernetes namespace to deploy to
        HELM_RELEASE_NAME = 'nodejs-release'     // The name of the Helm release
        HELM_CHART_PATH = './sample-nodejs-helm'  // Path to your Helm chart within the repo
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'Jenkins-cd', url: 'https://github.com/mustafizEnosis/helm-chart-prac.git' // Replace with your repo URL
            }
        }

        stage('Lint Helm Chart') {
            steps {
                script {
                    echo "Linting Helm chart at ${HELM_CHART_PATH}"
                    sh "helm lint ${HELM_CHART_PATH}"
                }
            }
        }

        stage('Helm Deploy') {
            steps {
                script {
                    echo "Deploying Helm chart ${HELM_CHART_PATH} as release ${HELM_RELEASE_NAME} in namespace ${KUBERNETES_NAMESPACE}"
                    // The --set flag overrides values in values.yaml
                    sh "helmfile sync"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    echo "Checking Helm release status for ${HELM_RELEASE_NAME} in namespace ${KUBERNETES_NAMESPACE}"

                    def maxAttempts = 18 // 3 minutes / 10 seconds per attempt = 18 attempts
                    def currentAttempt = 0
                    def verificationSucceeded = false

                    timeout(time: 3, unit: 'MINUTES') { // Outer timeout for the entire verification block
                        while (currentAttempt < maxAttempts && !verificationSucceeded) {
                            currentAttempt++
                            echo "Attempt ${currentAttempt}/${maxAttempts}: Checking Helm release status..."

                            def helmStatusOutput = sh(
                                script: "helm status ${HELM_RELEASE_NAME} --namespace ${KUBERNETES_NAMESPACE}",
                                returnStdout: true,
                                returnStatus: true
                            )

                            if (helmStatusOutput.status != 0) {
                                // This indicates a problem running the command (e.g., release not found, auth issue)
                                // If the command itself fails, we should fail the pipeline immediately,
                                // as retrying likely won't fix a command execution issue.
                                error "ERROR: 'helm status' command failed with exit code ${helmStatusOutput.status}. Output: ${helmStatusOutput.stdout}"
                            }

                            def releaseStatus = ''
                            def outputLines = helmStatusOutput.stdout.readLines()
                            for (def line : outputLines) {
                                if (line.trim().startsWith('STATUS:')) {
                                    releaseStatus = line.split(':')[1].trim()
                                    break
                                }
                            }

                            echo "Helm release '${HELM_RELEASE_NAME}' status: ${releaseStatus}"

                            switch (releaseStatus) {
                                case 'deployed':
                                    echo "Helm release '${HELM_RELEASE_NAME}' successfully deployed!"
                                    verificationSucceeded = true // Mark as success to exit loop
                                    break
                                case 'failed':
                                    error "Helm release '${HELM_RELEASE_NAME}' deployment FAILED! Check logs and application status."
                                    break
                                case 'superseded':
                                    // This is usually fine after an upgrade. You can decide to treat this as success,
                                    // or continue waiting if you expect 'deployed' after 'superseded' eventually.
                                    // For simplicity, we'll continue waiting unless it hits max attempts.
                                    echo "Helm release '${HELM_RELEASE_NAME}' superseded. Waiting for 'deployed' status."
                                    break
                                case 'pending-install':
                                case 'pending-upgrade':
                                case 'pending-rollback':
                                    echo "Helm release '${HELM_RELEASE_NAME}' is in a pending state (${releaseStatus}). Retrying..."
                                    break
                                case 'unknown':
                                    error "Helm release '${HELM_RELEASE_NAME}' is in an unknown state. This indicates a serious issue."
                                    break
                                default:
                                    // Handle any other unexpected statuses
                                    echo "Helm release '${HELM_RELEASE_NAME}' is in an unhandled status: ${releaseStatus}. Retrying..."
                                    break
                            }

                            if (!verificationSucceeded) {
                                sleep 10 // Wait 10 seconds before the next attempt
                            }
                        }
                    } // End of timeout block

                    if (!verificationSucceeded) {
                        error "Timeout: Helm release '${HELM_RELEASE_NAME}' did not reach 'deployed' status within 3 minutes."
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Cleaning up Docker login..."
                sh "docker logout ${DOCKER_REGISTRY}"
            }
        }
        success {
            echo "Pipeline finished successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}