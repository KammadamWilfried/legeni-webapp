// Pipeline Multibranch : tests -> build -> acceptance -> push
pipeline {
    agent any

    parameters {
        string(
            name: 'DOCKERHUB_ID',
            defaultValue: 'wilfriedkammadam',
            description: 'Votre identifiant Docker Hub'
        )
    }

    environment {
        IMAGE = "${params.DOCKERHUB_ID}/legeni-webapp"

        VERSION = "${env.BRANCH_NAME == 'main' ? '1.0.' + env.BUILD_NUMBER : env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-') + '-' + env.BUILD_NUMBER}"

        TEST_CONTAINER = "webapp-ci-${env.BUILD_NUMBER}"
        CI_NETWORK = "ci-net-${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Tests unitaires') {
            steps {
                echo 'Exécution des tests unitaires...'
                sh 'docker build --target test -t legeni-webapp:test .'
            }
        }

        stage('Build image') {
            steps {
                echo "Construction de l'image ${IMAGE}:${VERSION}"

                sh '''
                    docker build \
                        --build-arg APP_VERSION=$VERSION \
                        -t $IMAGE:$VERSION \
                        .
                '''
            }
        }

        stage("Test d'acceptance") {
            steps {
                sh '''
                    docker network create $CI_NETWORK

                    docker run -d \
                        --name $TEST_CONTAINER \
                        --network $CI_NETWORK \
                        $IMAGE:$VERSION

                    docker run --rm \
                        --network $CI_NETWORK \
                        curlimages/curl:8.11.1 \
                        -sf --retry 10 --retry-connrefused --retry-delay 1 \
                        http://$TEST_CONTAINER:5000/health

                    docker run --rm \
                        --network $CI_NETWORK \
                        curlimages/curl:8.11.1 \
                        -sf http://$TEST_CONTAINER:5000/ \
                        | grep -q "Version $VERSION"
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
                            -u "$DH_USER" \
                            --password-stdin

                        docker push $IMAGE:$VERSION
                        docker tag $IMAGE:$VERSION $IMAGE:latest
                        docker push $IMAGE:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f $TEST_CONTAINER || true
                docker network rm $CI_NETWORK || true
            '''
        }

        success {
            echo "Image ${IMAGE}:${VERSION} construite et validée."
        }

        failure {
            echo "Échec du pipeline : consultez la console."
        }
    }
}
