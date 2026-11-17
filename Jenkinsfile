pipeline {
    agent any

    parameters {
        string(
            name: 'DOCKERHUB_ID',
            defaultValue: 'wilfriedkammadam',
            description: 'Identifiant Docker Hub'
        )
    }

    environment {
        IMAGE = "${params.DOCKERHUB_ID}/legeni-webapp"
        VERSION = "${env.BRANCH_NAME == 'main' ? '1.0.' + env.BUILD_NUMBER : env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-') + '-' + env.BUILD_NUMBER}"
        COMPOSE = "docker compose -f deploy/docker-compose.deploy.yml"
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Tests unitaires') {
            steps {
                echo 'Exécution des tests, du lint et du contrôle de couverture...'

                sh '''
                    docker build --target test -t legeni-webapp:test .

                    mkdir -p reports

                    TEST_CONTAINER="legeni-tests-${BUILD_TAG}"

                    docker rm -f "$TEST_CONTAINER" 2>/dev/null || true

                    set +e
                    docker run --name "$TEST_CONTAINER" \
                        legeni-webapp:test \
                        sh -c "mkdir -p /reports && pytest -v \
                            --cov=app \
                            --cov-report=term-missing \
                            --cov-report=xml:/reports/coverage.xml \
                            --cov-fail-under=80 \
                            --junitxml=/reports/junit.xml"

                    TEST_STATUS=$?
                    set -e

                    docker cp "$TEST_CONTAINER:/reports/." "$WORKSPACE/reports/"

                    docker rm "$TEST_CONTAINER"

                    exit "$TEST_STATUS"
                '''

                junit 'reports/junit.xml'

                archiveArtifacts(
                    artifacts: 'reports/coverage.xml',
                    fingerprint: true
                )
            }
        }

        stage('Build image') {
            steps {
                echo "Construction de ${IMAGE}:${VERSION}"

                sh '''
                    docker build \
                        --build-arg APP_VERSION=$VERSION \
                        -t $IMAGE:$VERSION .
                '''
            }
        }

        stage('Analyse de sécurité Trivy') {
            steps {
                echo "Analyse de sécurité de ${IMAGE}:${VERSION}"

                sh '''
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                        --scanners vuln \
                        --severity CRITICAL \
                        --ignore-unfixed \
                        --exit-code 1 \
                        $IMAGE:$VERSION
                '''
            }
        }

        stage('Push Docker Hub') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login \
                            -u "$DH_USER" --password-stdin

                        docker push $IMAGE:$VERSION
                    '''
                }
            }
        }

        stage('Déploiement staging') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    string(
                        credentialsId: 'redis-password-staging',
                        variable: 'REDIS_PASSWORD'
                    )
                ]) {
                    sh '''
                        IMAGE_REF=$IMAGE:$VERSION \
                        APP_ENV=staging \
                        APP_COLOR='#bf8700' \
                        PORT=8081 \
                        REDIS_PASSWORD="$REDIS_PASSWORD" \
                        $COMPOSE -p legeni-staging up -d --wait
                    '''
                }
            }
        }

        stage('Test staging') {
            when {
                branch 'main'
            }
            steps {
                sh './deploy/smoke-test.sh legeni-staging "$VERSION"'
            }
        }

        stage('Validation humaine') {
            when {
                branch 'main'
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Staging validé. Autoriser le déploiement de ${VERSION} en production ?",
                          ok: 'Déployer en production'
                }
            }
        }

        stage('Déploiement et validation production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    env.PREVIOUS_IMAGE = sh(
                        script: '''
                            docker inspect -f '{{.Config.Image}}' legeni-prod-web-1
                        ''',
                        returnStdout: true
                    ).trim()

                    echo "Ancienne image de production : ${env.PREVIOUS_IMAGE}"
                }

                withCredentials([
                    string(
                        credentialsId: 'redis-password-prod',
                        variable: 'REDIS_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "Déploiement de la version $VERSION en production"

                        IMAGE_REF=$IMAGE:$VERSION \
                        APP_ENV=prod \
                        APP_COLOR='#1a7f37' \
                        PORT=8082 \
                        REDIS_PASSWORD="$REDIS_PASSWORD" \
                        $COMPOSE -p legeni-prod up -d --wait
                    '''

                    script {
                        int resultat = sh(
                            script: './deploy/smoke-test.sh legeni-prod "$VERSION"',
                            returnStatus: true
                        )

                        if (resultat != 0) {
                            echo 'Smoke test KO : début du rollback.'

                            if (env.PREVIOUS_IMAGE?.trim()) {
                                sh '''
                                    IMAGE_REF="$PREVIOUS_IMAGE" \
                                    APP_ENV=prod \
                                    APP_COLOR='#1a7f37' \
                                    PORT=8082 \
                                    REDIS_PASSWORD="$REDIS_PASSWORD" \
                                    $COMPOSE -p legeni-prod up -d --wait
                                '''

                                echo "Retour à l'image ${env.PREVIOUS_IMAGE}"
                            } else {
                                echo 'Aucune ancienne image disponible pour le rollback.'
                            }

                            error('Échec du smoke test production.')
                        }

                        echo 'Production validée avec succès !'
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline réussi : ${IMAGE}:${VERSION}"
        }

        failure {
            echo 'Échec du pipeline. Consultez la console Jenkins.'
        }

        always {
            sh 'docker image prune -f || true'
        }
    }
}
