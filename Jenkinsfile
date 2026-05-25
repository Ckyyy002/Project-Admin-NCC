pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    triggers {
        githubPush()
    }

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to checkout')
        string(name: 'GIT_URL', defaultValue: 'https://github.com/irfansandyy/final-project-ncc-kel3.git', description: 'Repository URL')
        string(name: 'GIT_CREDENTIALS_ID', defaultValue: '', description: 'Optional Jenkins credentials ID for private repos')
        string(name: 'DEPLOY_USER', defaultValue: 'ubuntu', description: 'SSH user on the target VPS')
        string(name: 'DEPLOY_DIR', defaultValue: '/opt/app', description: 'Remote directory where the app lives on the VPS')
    }

    environment {
        SONARQUBE_ENV    = 'SonarQube'
        PROJECT_KEY      = 'tugas-ncc-irfansandy-backend'
        PROJECT_NAME     = 'tugas-ncc-irfansandy-fullstack'
        GO_DIR           = 'backend'
        FE_DIR           = 'frontend'
        GOFLAGS          = '-buildvcs=false'
        SCANNER_HOME     = tool 'SonarQube Scanner'
        // Production domain
        DEPLOY_HOST      = 'llama-chat.my.id'
        // Jenkins credential ID (SSH Username with private key) for the VPS
        DEPLOY_SSH_CREDS = 'deploy-ssh-key'
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    try {
                        deleteDir()
                    } catch (Exception cleanupErr) {
                        echo "Workspace cleanup failed, fixing ownership/permissions and retrying deleteDir()"
                        sh '''
                            set -e
                            docker run --rm -u root -v "${WORKSPACE}:/workspace" alpine:3.21 \
                                sh -c 'chown -R 1000:1000 /workspace || true; chmod -R u+rwX /workspace || true'
                        '''
                        deleteDir()
                    }

                    if (params.GIT_CREDENTIALS_ID?.trim()) {
                        git branch: params.GIT_BRANCH,
                            url: params.GIT_URL,
                            credentialsId: params.GIT_CREDENTIALS_ID
                    } else {
                        git branch: params.GIT_BRANCH,
                            url: params.GIT_URL
                    }
                }
            }
        }

        stage('Setup') {
            agent {
                docker {
                    image 'golang:1.23-bookworm'
                    args '''-e HOME=/tmp \
                            -e GOCACHE=/tmp/go-cache \
                            -e GOPATH=/tmp/go \
                            -v /var/jenkins_home/tools:/var/jenkins_home/tools'''
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    git config --global --add safe.directory "${WORKSPACE}"
                    cd "${GO_DIR}"
                    go version
                    go mod download
                '''
            }
        }

        stage('Backend Build & Test') {
            parallel {
                stage('Build') {
                    agent {
                        docker {
                            image 'golang:1.23-bookworm'
                            args '''-e HOME=/tmp \
                                    -e GOCACHE=/tmp/go-cache \
                                    -e GOPATH=/tmp/go'''
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            set -e
                            cd "${GO_DIR}"
                            go build -v ./...
                        '''
                    }
                }

                stage('Test') {
                    agent {
                        docker {
                            image 'golang:1.23-bookworm'
                            args '''-e HOME=/tmp \
                                    -e GOCACHE=/tmp/go-cache \
                                    -e GOPATH=/tmp/go'''
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            set -e
                            cd "${GO_DIR}"
                            go test ./... -v -coverprofile=coverage.out
                        '''
                    }
                }
            }
        }

        stage('Frontend Install') {
            agent {
                docker {
                    image 'node:22-bookworm'
                    args '-e HOME=/tmp'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    cd "${FE_DIR}"
                    if [ -f package-lock.json ]; then
                        npm ci
                    else
                        npm install
                    fi
                '''
            }
        }

        stage('Frontend Lint') {
            agent {
                docker {
                    image 'node:22-bookworm'
                    args '-e HOME=/tmp'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    cd "${FE_DIR}"
                    if npx --yes next --help 2>&1 | grep -Eq '(^|[[:space:]])lint([[:space:]]|$)'; then
                        npm run lint
                    else
                        echo "next lint is not available on this Next.js version, skipping lint stage."
                    fi
                '''
            }
        }

        stage('Frontend Build') {
            agent {
                docker {
                    image 'node:22-bookworm'
                    args '-e HOME=/tmp'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    set -e
                    cd "${FE_DIR}"
                    npm run build
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                        set -e
                        "${SCANNER_HOME}"/bin/sonar-scanner \
                            -Dsonar.projectKey="${PROJECT_KEY}" \
                            -Dsonar.projectName="${PROJECT_NAME}" \
                            -Dsonar.sources=backend,frontend \
                            -Dsonar.tests=backend,frontend \
                            -Dsonar.test.inclusions=backend/**/*_test.go,frontend/**/*.test.js,frontend/**/*.test.ts,frontend/**/*.spec.js,frontend/**/*.spec.ts \
                            -Dsonar.exclusions=frontend/.next/**,frontend/node_modules/** \
                            -Dsonar.go.coverage.reportPaths=backend/coverage.out
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 20, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        // DEPLOY STAGE
        // Deploys to https://llama-chat.my.id — runs on the main branch only.
        //
        // Pre-requisites (one-time setup):
        //   1. Jenkins credential "deploy-ssh-key" → SSH Username with private key
        //      whose public key is in ~/.ssh/authorized_keys on the VPS.
        //   2. The VPS already has the repo cloned at DEPLOY_DIR with a .env file.
        //   3. "SSH Agent" Jenkins plugin installed.
        // ─────────────────────────────────────────────────────────────────────
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: [env.DEPLOY_SSH_CREDS]) {
                    sh """
                        set -e

                        REMOTE="${params.DEPLOY_USER}@${DEPLOY_HOST}"
                        DIR="${params.DEPLOY_DIR}"

                        echo "==> [1/3] Pulling latest code on llama-chat.my.id"
                        ssh -o StrictHostKeyChecking=no \$REMOTE \\
                            "cd \$DIR && git pull origin main"

                        echo "==> [2/3] Rebuilding & restarting containers"
                        ssh -o StrictHostKeyChecking=no \$REMOTE \\
                            "cd \$DIR && docker compose --env-file .env up -d --build --remove-orphans"

                        echo "==> [3/3] Health check — https://llama-chat.my.id/health"
                        ssh -o StrictHostKeyChecking=no \$REMOTE \\
                            'timeout 120 sh -c '"'"'until curl -ks https://llama-chat.my.id/health | grep -q ok; do echo "Waiting..."; sleep 5; done'"'"' && echo "✓ llama-chat.my.id is healthy"'
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline sukses — https://llama-chat.my.id live!'
        }
        failure {
            echo 'Pipeline gagal'
        }
        always {
            archiveArtifacts artifacts: 'backend/coverage.out', allowEmptyArchive: true
            archiveArtifacts artifacts: 'frontend/.next/**', allowEmptyArchive: true
        }
    }
}
