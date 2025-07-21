pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_NAME_WITH_TAG', defaultValue: 'bibek87/dynamic_weather_app:latest', description: 'Docker image name and tag from the upstream CI build.')
    }

    environment {
        // Retrieve API key from Jenkins credentials, same as in CI
        OPENWEATHER_API_KEY = credentials('OPENWEATHER_API_KEY')
        CONTAINER_NAME = "dynamicweatherapp_cont_cd" // Name for the deployed container
        APP_PORT = 5000 // Port the application listens on inside the container and on the host
        HEALTH_CHECK_ENDPOINT = "/api/health"
    }

    stages {
        stage('Checkout CD Repo') {
            steps {
                echo "---Checking out CD deployment repository---"
                // This Jenkinsfile is already checked out as part of the pipeline.
                // Just ensuring the workspace is clean and files are present.
                cleanWs() // Clean the workspace before checkout
                checkout scm // Checkout the CD repo itself
            }
        }

        stage('Update Deployment Manifest') {
            steps {
                echo "---Updating docker-compose.yml with new image tag: ${params.IMAGE_NAME_WITH_TAG}---"
                // Use sed to replace the placeholder tag with the actual image tag
                sh "sed -i 's|image: bibek87/dynamic_weather_app:PLACEHOLDER_TAG|image: ${params.IMAGE_NAME_WITH_TAG}|g' docker-compose.yml"
                sh "cat docker-compose.yml" // Print content to verify the update
            }
        }

        stage('Commit and Push Deployment Config') {
            steps {
                script {
                    // Configure Git user (important for commits)
                    sh 'git config user.email "bibek.tamrakar+pipeline@msn.com"' // Use a dedicated Jenkins email
                    sh 'git config user.name "tba87_jenkins"' // Use a dedicated Jenkins name

                    // Add the modified file
                    sh 'git add docker-compose.yml' // Or git add . if you modified multiple files

                    // Check if there are actual changes before committing
                    def changes = sh(script: 'git status --porcelain', returnStdout: true).trim()
                    if (changes) {
                        echo "Committing updated docker-compose.yml with new image tag..."
                        sh "git commit -m 'Deploy: Update image tag to ${params.IMAGE_NAME_WITH_TAG}'"
                        withCredentials([usernamePassword(credentialsId: 'Github_token', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                            sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@https://github.com/tba87/dynamicweatherapp_CD.git HEAD" // Or use SSH URL if configured
                        }
                        echo "Changes pushed to GitHub."
                    } else {
                        echo "No changes detected in docker-compose.yml to commit."
                    }
                }
            }
        }

        stage('Deploy Application') {
            steps {
                echo "---Deploying application using Docker Compose---"
                // Ensure docker compose is installed on the Jenkins agent
                // Stop and remove any previously deployed container for a clean redeployment
                sh "docker compose down --remove-orphans || true" // --remove-orphans cleans up volumes not defined in compose
                // Pull the new image from Docker Hub (ensures latest image from CI is used)
                sh "docker compose pull"
                // Start the application in detached mode (-d)
                sh "OPENWEATHER_API_KEY=${OPENWEATHER_API_KEY} docker compose up -d"
                echo "Deployment initiated. Waiting for application to become ready..."
                sleep(time: 10, unit: 'SECONDS') // Give the container some time to start up
            }
        }

        stage('Health Check') {
            steps {
                echo "---Performing health check on deployed application---"
                script {
                    def maxRetries = 15
                    def retryCount = 0
                    def appHealthy = false
                    while (retryCount < maxRetries && !appHealthy) {
                        try {
                            // Use curl to hit the health endpoint. --fail makes it exit with error on HTTP error codes.
                            sh "curl --fail --silent http://localhost:${APP_PORT}${HEALTH_CHECK_ENDPOINT}"
                            appHealthy = true
                            echo "Application is healthy and running!"
                        } catch (e) {
                            echo "Health check failed. Retrying in 5 seconds... (${++retryCount}/${maxRetries})"
                            sleep(time: 5, unit: 'SECONDS')
                        }
                    }
                    if (!appHealthy) {
                        error("Application failed health check after ${maxRetries} retries. Deployment considered failed.")
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "---Post-deployment tasks completed.---"
                // For a persistent deployment on the Jenkins server, we generally want the app to keep running.
                // So, we don't 'docker compose down' here. The 'down' at the start of 'Deploy Application'
                // stage handles cleanup of the *previous* version before deploying the new one.
                // If this was a temporary test deployment that needs to be torn down *after* verification,
                // you would place `sh "docker compose down"` here.
            }
        }
        failure {
            echo "---CD Pipeline failed. Check logs for details.---"
            // Optional: Add specific cleanup for failed deployments if the app shouldn't remain running.
            // sh "docker compose down || true"
        }
    }
}