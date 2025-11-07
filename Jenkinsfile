pipeline {
    agent {
        label 'docker-agent'
    }
    
    environment {
        APP_NAME = 'microblog'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        REGISTRY = 'ghcr.io'
        GITHUB_CREDS = credentials('github-token')
        DOCKER_IMAGE = "${REGISTRY}/${GITHUB_CREDS_USR}/${APP_NAME}:${IMAGE_TAG}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
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
                    . venv/bin/activate
                    pip install -q bandit[toml]
                    bandit -r app/ -f json -o bandit-report.json || true
                    bandit -r app/ -ll || true
                '''
            }
        }
        
        stage('Dockerfile Linting') {
            steps {
                sh 'docker run --rm -i hadolint/hadolint:latest-alpine < Dockerfile | tee hadolint-report.txt || true'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} ."
                sh "docker tag ${DOCKER_IMAGE} ${REGISTRY}/${GITHUB_CREDS_USR}/${APP_NAME}:latest"
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
        
        stage('Container Vulnerability Scan') {
            steps {
                sh """
                    docker run --rm \\
                        -e DOCKER_HOST=tcp://172.17.0.1:2375 \\
                        aquasec/trivy:latest image \\
                        --format json \\
                        --severity HIGH,CRITICAL \\
                        ${DOCKER_IMAGE} || true
                """
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
                        docker push ${REGISTRY}/${GITHUB_CREDS_USR}/${APP_NAME}:latest
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                try {
                    archiveArtifacts artifacts: 'hadolint-report.txt,test-results.xml,bandit-report.json,trivy-report.json', 
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