pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        skipDefaultCheckout()
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        DOCKER_IMAGE = 'stenythomas/villa-app'
        CI_IMAGE = "villa-app:ci-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Environment') {
            steps {
                script {
                    env.CI_DB_PASSWORD = UUID.randomUUID().toString()
                    env.CI_ROOT_PASSWORD = UUID.randomUUID().toString()
                    env.CI_SECRET_KEY = UUID.randomUUID().toString()
                }

                sh '''
                    rm -rf security-reports zap-output
                    mkdir -p security-reports zap-output

                    python3 -m venv .ci-venv
                    .ci-venv/bin/pip install --upgrade pip
                    .ci-venv/bin/pip install bandit pip-audit
                '''
            }
        }

        stage('SAST - Bandit') {
            steps {
                sh '''
                    .ci-venv/bin/bandit -r villaproject admin_app builder_app user_app \
                        -f html \
                        -o security-reports/sast-report.html
                '''
            }
        }

        stage('Dependency Scan') {
            steps {
                sh '''
                    .ci-venv/bin/pip-audit \
                        -r requirements.txt \
                        -f json \
                        -o security-reports/dependency-report.json
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t "$CI_IMAGE" .
                '''
            }
        }

        stage('Start Application') {
            steps {
                sh '''
                    docker compose \
                        -p villa-ci \
                        -f docker-compose.ci.yml \
                        up -d --wait

                    docker compose \
                        -p villa-ci \
                        -f docker-compose.ci.yml ps
                '''
            }
        }

        stage('Application Testing') {
            steps {
                sh '''
                    docker compose \
                        -p villa-ci \
                        -f docker-compose.ci.yml \
                        exec -T web python manage.py check

                    docker compose \
                        -p villa-ci \
                        -f docker-compose.ci.yml \
                        exec -T web python manage.py migrate --check

                    curl --fail --silent --show-error \
                        --retry 10 \
                        --retry-connrefused \
                        --retry-delay 3 \
                        http://127.0.0.1:8011/ \
                        -o /dev/null
                '''
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                script {
                    sh 'chmod 777 zap-output'

                    def zapStatus = sh(
                        script: '''
                            docker run --rm --network host \
                              -v "$WORKSPACE/zap-output:/zap/wrk/:rw" \
                              ghcr.io/zaproxy/zaproxy:stable \
                              zap-baseline.py \
                              -t http://127.0.0.1:8011 \
                              -m 1 \
                              -r dast-report.html \
                              -J dast-report.json
                        ''',
                        returnStatus: true
                    )

                    sh '''
                        test -s zap-output/dast-report.html
                        test -s zap-output/dast-report.json
                        cp zap-output/dast-report.* security-reports/
                    '''

                    if (zapStatus == 2) {

                        archiveArtifacts(
                            artifacts: 'security-reports/dast-report.*'
                        )

                        input(
                            message: 'Review the DAST report and approve remaining warnings for local deployment.',
                            ok: 'Approve Warnings'
                        )

                    } else if (zapStatus != 0) {
                        error("ZAP failed with exit code ${zapStatus}")
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'Dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        export DOCKER_CONFIG=$(mktemp -d)
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        echo "$DOCKER_PASSWORD" | \
                            docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin

                        docker tag "$CI_IMAGE" \
                            "$DOCKER_IMAGE:$BUILD_NUMBER"

                        docker tag "$CI_IMAGE" \
                            "$DOCKER_IMAGE:latest"

                        docker push "$DOCKER_IMAGE:$BUILD_NUMBER"
                        docker push "$DOCKER_IMAGE:latest"

                        docker logout
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'security-reports/*',
                allowEmptyArchive: true
            )

            sh '''
                if [ -n "$CI_DB_PASSWORD" ]; then
                    docker compose \
                        -p villa-ci \
                        -f docker-compose.ci.yml \
                        down -v --remove-orphans
                fi
            '''
        }

        success {
            echo 'Villa DevSecOps Pipeline Successful'
        }

        failure {
            echo 'Villa Pipeline Failed'
        }
    }
}
