#!/usr/bin/env groovy

pipeline {
    agent any

    tools {
        nodejs 'Node24'
    }

    environment {
        APP_DIR = 'app'
        DOCKER_IMAGE_REPOSITORY = 'younghadiz/nodejs-jenkins-cicd'
        GITHUB_REPOSITORY_HOST_PATH = 'github.com/younghadiz/nodejs-jenkins-cicd-pipeline.git'
    }

    options {
        buildDiscarder(logRotator(
            numToKeepStr: '20',
            artifactNumToKeepStr: '10'
        ))

        disableConcurrentBuilds()

        timestamps()

        timeout(
            time: 30,
            unit: 'MINUTES'
        )

        skipDefaultCheckout(false)
    }

    stages {
        stage('Validate Environment') {
            steps {
                sh '''
                    set -eu

                    echo "Node version:"
                    node --version

                    echo "npm version:"
                    npm --version

                    echo "Docker version:"
                    docker --version

                    echo "Git version:"
                    git --version
                '''
            }
        }

        stage('Increment Version') {
            steps {
                dir(env.APP_DIR) {
                    script {
                        /*
                         * --no-git-tag-version is essential in Jenkins.
                         * It changes package.json and package-lock.json without
                         * creating an npm-generated Git commit or Git tag.
                         */
                        sh 'npm version minor --no-git-tag-version'

                        def packageJson = readJSON file: 'package.json'
                        env.APP_VERSION = packageJson.version
                        env.IMAGE_TAG = "${env.APP_VERSION}-${env.BUILD_NUMBER}"

                        echo "Application version: ${env.APP_VERSION}"
                        echo "Docker image tag: ${env.IMAGE_TAG}"
                        echo "Full image: ${env.DOCKER_IMAGE_REPOSITORY}:${env.IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir(env.APP_DIR) {
                    sh '''
                        set -eu
                        npm ci
                    '''
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir(env.APP_DIR) {
                    sh '''
                        set -eu
                        npm test -- --runInBand
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build \
                      --pull \
                      --tag "${DOCKER_IMAGE_REPOSITORY}:${IMAGE_TAG}" \
                      --tag "${DOCKER_IMAGE_REPOSITORY}:latest" \
                      .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "${DOCKER_TOKEN}" |
                          docker login \
                            --username "${DOCKER_USERNAME}" \
                            --password-stdin

                        set -x
                        docker push "${DOCKER_IMAGE_REPOSITORY}:${IMAGE_TAG}"
                        docker push "${DOCKER_IMAGE_REPOSITORY}:latest"
                        set +x

                        docker logout
                    '''
                }
            }
        }

        stage('Commit Version Update') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins CI"

                        git remote set-url origin \
                          "https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@${GITHUB_REPOSITORY_HOST_PATH}"

                        set -x
                        git status --short

                        git add \
                          "${APP_DIR}/package.json" \
                          "${APP_DIR}/package-lock.json"

                        if git diff --cached --quiet; then
                          echo "No npm version files changed; nothing to commit."
                          exit 0
                        fi

                        git commit -m "ci: bump Node.js application version [jenkins]"
                        git push origin "HEAD:${BRANCH_NAME}"
                        set +x
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"

            sh '''
                docker logout >/dev/null 2>&1 || true
            '''

            deleteDir()
        }

        success {
            echo "Successfully published ${env.DOCKER_IMAGE_REPOSITORY}:${env.IMAGE_TAG}"
        }

        failure {
            echo 'Pipeline failed. Review the failed stage and console output.'
        }
    }
}