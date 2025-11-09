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
                            venv_dir=".venv-tests"
                            python3 -m venv "$venv_dir"
                            . "$venv_dir/bin/activate"
                            python -m pip install -q --upgrade pip
                            python -m pip install -q -r requirements.txt
                            python -m pip install -q pytest
                            set +e
                            python -m pytest -v --junitxml=test-results.xml
                            exit_code=$?
                            set -e
                            if [ "$exit_code" -eq 5 ]; then
                              echo "No tests collected; treating as success."
                              exit 0
                            fi
                            exit "$exit_code"
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'test-results.xml'
                        }
                    }
                }
                
                stage('SAST - Security Scanning') {
                    steps {
                        sh '''
                            venv_dir=".venv-sast"
                            python3 -m venv "$venv_dir"
                            . "$venv_dir/bin/activate"
                            python -m pip install -q --upgrade pip
                            python -m pip install -q "bandit[toml]"
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
                                    # Ensure no leftover container blocks the name
                                    docker rm -f ${testContainer} || true
                                    
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
                    sh '''
                        echo "$GITHUB_CREDS_PSW" | docker login "$REGISTRY" -u "$GITHUB_CREDS_USR" --password-stdin
                        docker push "$DOCKER_IMAGE"
                        docker push "$IMAGE_REPO:$GIT_COMMIT_SHORT"
                    '''
                }
            }
        }
        
        stage('Update Helm Chart Repository') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh '''
                        set -eu
                        # Clone helm chart repository
                        git clone "https://$GITHUB_CREDS_USR:$GITHUB_CREDS_PSW@github.com/cloud-infra-assignment/helm-microblog.git" helm-chart-repo
                        cd helm-chart-repo
                        # Ensure we are on main
                        git fetch origin main
                        git checkout -B main origin/main
                        
                        git config user.email "artyom.k.devops@posteo.net"
                        git config user.name "Artyom K"
                        
                        # Find values.yaml intelligently
                        FILE=""
                        if [ -f "microblog/values.yaml" ]; then
                          FILE="microblog/values.yaml"
                        elif [ -f "values.yaml" ]; then
                          FILE="values.yaml"
                        else
                          # look for common helm chart layouts
                          FILE="$(git ls-files | grep -E '(^|/)(charts|helm)/[^/]+/values\\.yaml$' | head -n1 || true)"
                        fi
                        if [ -z "$FILE" ] || [ ! -f "$FILE" ]; then
                          echo "ERROR: Could not find values.yaml in repo. Checked: microblog/values.yaml, values.yaml, charts/*/values.yaml, helm/*/values.yaml"
                          exit 1
                        fi
                        
                        # Update values.yaml with new image using sed (portable GNU sed)
                        # Update tag
                        sed -i "s|^[[:space:]]*tag:.*|tag: \\\"$IMAGE_TAG\\\"|" "$FILE"
                        # Update repository
                        sed -i "s|^[[:space:]]*repository:.*|repository: $REGISTRY/$GITHUB_CREDS_USR/$APP_NAME|" "$FILE"
                        
                        # Commit and push
                        git add "$FILE"
                        git commit -m "Update image to $REGISTRY/$GITHUB_CREDS_USR/$APP_NAME:$IMAGE_TAG" || true
                        git push origin main
                    '''
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