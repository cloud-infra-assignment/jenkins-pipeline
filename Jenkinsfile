pipeline {
    agent {
        label 'docker-agent'
    }
    
    environment {
        APP_NAME = 'microblog'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        REGISTRY = 'ghcr.io'
        GITHUB_CREDS = credentials('github-token')
        IMAGE_REPO = "${REGISTRY}/${GITHUB_CREDS_USR}/${APP_NAME}"
        DOCKER_IMAGE = "${IMAGE_REPO}:${IMAGE_TAG}"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    def gitCommitShort = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.GIT_COMMIT_SHORT = gitCommitShort
                    sh "docker build -t ${DOCKER_IMAGE} ."
                    sh "docker tag ${DOCKER_IMAGE} ${IMAGE_REPO}:${gitCommitShort}"
                }
            }
        }
        
        // All checks and validation in parallel
        stage('Checks & Validation') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh '''
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install -q --upgrade pip
                            pip install -q -r requirements.txt
                            pip install -q pytest
                            python -m pytest tests.py -v --junitxml=test-results.xml
                        '''
                    }
                    post {
                        always {
                            junit 'test-results.xml' 
                        }
                    }
                }
                
                stage('SAST - Security Scanning') {
                    steps {
                        sh '''
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install -q bandit[toml]
                            bandit -r app/ -f json -o bandit-report.json || true
                            bandit -r app/ -ll || true
                        '''
                    }
                }
                
                stage('Secret Scanning') {
                    steps {
                        sh 'docker run --rm -v ${WORKSPACE}:/repo trufflesecurity/trufflehog:latest filesystem /repo --json > trufflehog-report.json || true'
                    }
                }
                
                stage('Dockerfile Linting') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint:latest-alpine < Dockerfile | tee hadolint-report.txt || true'
                    }
                }
                
                stage('Container Smoke Test') {
                    steps {
                        script {
                            def testContainer = "${APP_NAME}-test-${BUILD_NUMBER}"
                            try {
                                sh """
                                    docker run -d --name ${testContainer} \\
                                        -e SECRET_KEY=test -e DATABASE_URL=sqlite:///test.db ${DOCKER_IMAGE}
                                    
                                    # Retry loop: wait up to 30 seconds for app to start
                                    for i in \$(seq 1 15); do
                                        if docker exec ${testContainer} python3 -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/')" 2>/dev/null; then
                                            echo "Smoke test passed on attempt \$i"
                                            exit 0
                                        fi
                                        echo "Attempt \$i failed, retrying..."
                                        sleep 2
                                    done
                                    
                                    echo "Smoke test failed after 15 attempts"
                                    exit 1
                                """
                            } finally {
                                sh "docker rm -f ${testContainer} || true"
                            }
                        }
                    }
                }
                
                // Non-blocking: Reports on CVEs but does not fail the build
                stage('Container Vulnerability Scan') {
                    steps {
                        sh """
                            mkdir -p "${WORKSPACE}/.trivy"
                            docker run --rm \\
                                -v /var/run/docker.sock:/var/run/docker.sock \\
                                -v "${WORKSPACE}/.trivy:/root/.cache/" \\
                                aquasec/trivy:latest image \\
                                --format json \\
                                --severity HIGH,CRITICAL \\
                                --no-progress \\
                                ${DOCKER_IMAGE} > trivy-report.json || true
                        """
                    }
                }
            }
        }
        
        stage('Push to Registry') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        echo ${GITHUB_CREDS_PSW} | docker login ${REGISTRY} -u ${GITHUB_CREDS_USR} --password-stdin
                        docker push ${DOCKER_IMAGE}
                        docker push ${IMAGE_REPO}:${GIT_COMMIT_SHORT}
                    """
                }
            }
        }
        
        stage('Update Helm Chart Repository') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh """
                        # Clone helm chart repository
                        git clone https://${GITHUB_CREDS_USR}:${GITHUB_CREDS_PSW}@github.com/cloud-infra-assignment/helm-microblog.git helm-chart-repo
                        cd helm-chart-repo
                        
                        git config user.email "artyom.k.devops@posteo.net"
                        git config user.name "Artyom K"
                        # Update values.yaml with new image using YAML-aware tool (yq)
                        docker run --rm -e IMAGE_REPO="${IMAGE_REPO}" -v "${WORKSPACE}/helm-chart-repo":/workdir -w /workdir mikefarah/yq:4 \\
                          e --inplace --expression '.image.repository = strenv(IMAGE_REPO)' microblog/values.yaml
                        docker run --rm -e IMAGE_TAG="${IMAGE_TAG}" -v "${WORKSPACE}/helm-chart-repo":/workdir -w /workdir mikefarah/yq:4 \\
                          e --inplace --expression '.image.tag = strenv(IMAGE_TAG)' microblog/values.yaml
                        
                        # Commit and push
                        git add microblog/values.yaml
                        git commit -m "Update image to ${REGISTRY}/${GITHUB_CREDS_USR}/${APP_NAME}:${IMAGE_TAG}" || true
                        git push origin main
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                try {
                    archiveArtifacts artifacts: 'hadolint-report.txt,test-results.xml,bandit-report.json,trufflehog-report.json,trivy-report.json', 
                        allowEmptyArchive: true
                } catch (Exception e) {
                    echo "Failed to archive artifacts: ${e.message}"
                }
            }
        }
        cleanup {
            cleanWs()
        }
    }
}