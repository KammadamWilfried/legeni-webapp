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
        COMPOSE = "docker compose -f deploy/docker-compose.deploy.yml"
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

                stage('Définir la version') {
            steps {
                script {
                    def baseVersion = readFile('VERSION').trim()

                    if (!(baseVersion ==~ /^[0-9]+\.[0-9]+\.[0-9]+$/)) {
                        error("Format de version invalide : ${baseVersion}")
                    }

                    env.VERSION = "${baseVersion}-${env.BUILD_NUMBER}"

                    echo "Version à construire : ${env.VERSION}"
                }
            }
        }
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
                echo "Déploiement Kubernetes staging : ${VERSION}"

                sh """
                    ansible-playbook ansible/deploy.yml \
                        -e target_namespace=staging \
                        -e image_tag=${VERSION} \
                        -e image_name=${IMAGE}
                """
            }
        }

        stage('Test staging') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    kubectl rollout status deployment/legeni-webapp \
                        -n staging --timeout=180s

                    kubectl get deployments -n staging
                """
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

        stage('Déploiement production') {
            when {
                branch 'main'
            }
            steps {
                echo "Déploiement Kubernetes production : ${VERSION}"

                sh """
                    ansible-playbook ansible/deploy.yml \\
                        -e target_namespace=prod \\
                        -e image_tag=${VERSION} \\
                        -e image_name=${IMAGE}
                """
            }
        }

        stage('Vérification production') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    kubectl rollout status deployment/legeni-webapp \\
                        -n prod --timeout=180s

                    kubectl get deployments -n prod
                """
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
