pipeline {
    agent any
    environment {
        MERKELY_API_TOKEN = credentials('merkely-api')
        GITHUB = credentials('github')
        MERKELY_CLI_VERSION = "1.4.1"
        MERKELY_PIPELINE = "docker-test"
        MERKELY_ENVIRONMENT = "staging-k8s"

        DOCKERHUB_PAT = 'dockerhub-pat'
        DOCKER_ORG = "ewelinawilkosz"
        IMAGE_NAME = "time-server"
        DOCKER_IMAGE_NAME = "$DOCKER_ORG/$IMAGE_NAME"
        DOCKER_IMAGE = ''
    }
    stages {
        stage('Prepare') {
            steps {
                sh(
                    label: 'Check enviroment variables',
                    script: """
                    echo "MERKELY_HOST:"
                    echo $MERKELY_HOST
                    echo "MERKELY_OWNER:"
                    echo $MERKELY_OWNER
                    echo "DOCKER_IMAGE_NAME:"
                    echo $DOCKER_IMAGE_NAME
                    """
                )
                sh(
                    label: 'Download client',
                    script: """
                    wget https://github.com/merkely-development/cli/releases/download/v${env.MERKELY_CLI_VERSION}/merkely_${env.MERKELY_CLI_VERSION}_linux_amd64.tar.gz
                    tar -xf merkely_${env.MERKELY_CLI_VERSION}_linux_amd64.tar.gz
                    """
                )
                sh(
                    label: 'Create a pipeline',
                    script: """
                    ./merkely pipeline declare --description "testing new cli from Jenkins" --template artifact,unit-test,pull-request
                    """
                )
                // Example with different evidence template
                // sh(
                //     label: 'Create a pipeline',
                //     script: """
                //     ./merkely pipeline declare --description "testing new cli from Jenkins" --template artifact,unit-test,pull-request
                //     """
                // )
                sh(
                    label: 'Create an environment',
                    script: """
                    ./merkely environment declare --name ${env.MERKELY_ENVIRONMENT} --environment-type K8S --description "Staging env for K8S deployments (test)"
                    """
                )
            }
        }
        stage('Build and report') {
            steps {
                sh(
                    label: 'Prepare content',
                    script: """
                    date > date-and-time.txt

                    """
                )
                script {
                    DOCKER_IMAGE = docker.build DOCKER_IMAGE_NAME
                    docker.withRegistry('', DOCKERHUB_PAT) {
                        DOCKER_IMAGE.push("$BUILD_NUMBER")
                        DOCKER_IMAGE.push('latest')
                    }
                }
                sh(
                    label: 'Report artifact creation',
                    script: """
                    ./merkely pipeline artifact report creation --artifact-type docker \
                        --build-url $BUILD_URL \
                        --commit-url ${GIT_URL}/${GIT_COMMIT} \
                        --git-commit ${GIT_COMMIT} \
                        $DOCKER_IMAGE_NAME:$BUILD_NUMBER
                    """
                )
            }
        }
        stage('Test and report') {
            steps {
                sh(
                    label: 'Report unit-test',
                    script: """
                    ./merkely pipeline artifact report evidence generic --artifact-type docker \
                        --build-url $BUILD_URL \
                        --evidence-type "unit-test"\
                        $DOCKER_IMAGE_NAME:$BUILD_NUMBER
                    """
                )
                // creating a branch and a PR to verify if PR reporting works as it should
                withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'DH_USERNAME', passwordVariable: 'DH_PASSWORD')]) {
                    sh(
                        label: 'Report pull request',
                        script: """
                        ./merkely pipeline artifact report evidence github-pullrequest --artifact-type docker \
                            --build-url $BUILD_URL \
                            --evidence-type "pull-request" \
                            --commit ${GIT_COMMIT} \
                            --github-org merkely-development \
                            --github-token $DH_PASSWORD \
                            --repository jenkins-demo \
                            $DOCKER_IMAGE_NAME:$BUILD_NUMBER
                        """
                    )   
                }
            }
        }

        stage('Approve') {
            environment {
                MERKELY_OLDEST_COMMIT = "${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT}"
                MERKELY_NEWEST_COMMIT = "${env.GIT_COMMIT}"
                MERKELY_DESCRIPTION = "Approval created by Jenkins job ${BUILD_URL}"
            }
            steps {
                sh(
                    label: 'Report approval',
                    script: """
                    ./merkely pipeline approval report --artifact-type docker $DOCKER_IMAGE_NAME:$BUILD_NUMBER
                    """
                )
            }
        }
        stage('Deploy') {
            steps {
                sh "sed -i \'s/$IMAGE_NAME/$IMAGE_NAME:$BUILD_NUMBER/g\' k8s/deployment.yaml"
                sh(
                    label: 'Report deployment',
                    script: """
                    cat k8s/deployment.yaml
                    gcloud container clusters get-credentials merkely-dev --region europe-west1 --project cellular-deck-259216
                    kubectl apply -f k8s/deployment.yaml
                    ./merkely pipeline deployment report --artifact-type docker --build-url $BUILD_URL $DOCKER_IMAGE_NAME:$BUILD_NUMBER

                    """
                )
            }
        }
        stage('Clean up') {
            steps {
                sh "docker rmi $DOCKER_IMAGE_NAME:$BUILD_NUMBER"
                sh "docker rmi $DOCKER_IMAGE_NAME:latest"
                sh "rm merkely_${env.MERKELY_CLI_VERSION}_linux_amd64.tar.gz*"
            }
        }
    }
}
