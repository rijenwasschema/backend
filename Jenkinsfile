def commitid

pipeline {
    agent any

	environment {
        DOCKER_REGISTRY = 'registry.sanderkoenders.nl'
		PRODUCT_NAME = 'archcry/korfbalbackend'
        COMPOSE_FILE_GIT = 'ssh://git@git.sanderkoenders.nl:7999/docker/docker-compose.git'
	}

    options {
        gitlabBuilds(builds: ['Initialize build', 'Build Docker image', 'Push Docker image to Registry', 'Deploy docker image'])
    }

    post {
        always {
            deleteDir();
        }
    }

    stages {
        stage('Initialize build') {
            steps {
                gitlabCommitStatus(name: STAGE_NAME) {
                    sh "git rev-parse --short HEAD > commitid"
                    script {
                        commitid = readFile('commitid').trim()
                    }
                    sh "rm commitid";
                    echo "Git commit hash: ${commitid}"
                }
            }
        }

        stage("Build Docker image") {
            steps {
                gitlabCommitStatus(name: STAGE_NAME) {
                    script {
                        docker.withRegistry("https://${DOCKER_REGISTRY}", 'svc-docker-registry') {
                            sh "docker build --pull --force-rm -t ${PRODUCT_NAME} ."
                        }
                    }
                }
            }
        }

        stage("Push Docker image to Registry") {
            steps {
                gitlabCommitStatus(name: STAGE_NAME) {
                    script {
                        docker.withRegistry("https://${DOCKER_REGISTRY}", 'svc-docker-registry') {
                            echo "Pushing ${PRODUCT_NAME} to ${DOCKER_REGISTRY}/${PRODUCT_NAME} tagged as '${commitid}' and 'latest'"

                            // Tag docker image with commit id and as latest
                            sh "docker tag ${PRODUCT_NAME} ${DOCKER_REGISTRY}/${PRODUCT_NAME}:latest"
                            sh "docker tag ${PRODUCT_NAME} ${DOCKER_REGISTRY}/${PRODUCT_NAME}:${commitid}"

                            // Push both tags to the registry
                            sh "docker push ${DOCKER_REGISTRY}/${PRODUCT_NAME}:latest"
                            sh "docker push ${DOCKER_REGISTRY}/${PRODUCT_NAME}:${commitid}"                        
                        }
                    }
                }
            }
        }

        stage("Deploy docker image") {
            steps {
                gitlabCommitStatus(name: STAGE_NAME) {
                    script {
                        withCredentials([usernamePassword(credentialsId: 'svc-docker-registry', usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASS')]) {
                            // Get name of service inside docker compose file (should match)
                            def dockerComposeName = PRODUCT_NAME.split("/")[1];

                            // Clone compose file repository and deploy product
                            sh "git clone --depth=1 ${COMPOSE_FILE_GIT}"
                            sh "docker login -u ${REGISTRY_USER} -p ${REGISTRY_PASS} ${DOCKER_REGISTRY}"
                            sh "docker-compose -f docker-compose/docker-compose.yml -p default pull ${dockerComposeName}"
                            sh "docker-compose -f docker-compose/docker-compose.yml -p default up --remove-orphans -d ${dockerComposeName}"
                        }
                    }
                }
            }
        }
    }
}
