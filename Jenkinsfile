pipeline {
    agent any

    parameters {
        choice(name: 'ARTIFACTORY', choices: ['cloud', 'selfhosted'],
               description: 'Which Artifactory to resolve dependencies through')
    }

    environment {
        // non-secret jfrog coordinates. needs to be updated if you use a different JFrog instance or repository.
        JFROG_URL   = "${params.ARTIFACTORY == 'selfhosted' ? 'http://localhost:8082' : 'https://trialp0kvcv.jfrog.io'}"
        JFROG_REPO  = "${params.ARTIFACTORY == 'selfhosted' ? 'petclinic-virtual' : 'petclinic-libs-snapshot'}"
        JFROG_CREDS = "${params.ARTIFACTORY == 'selfhosted' ? 'artifactory-selfhosted' : 'jfrog-cloud'}"
        DOCKER_NET  = "${params.ARTIFACTORY == 'selfhosted' ? '--network host' : ''}"
    }

    stages {
        stage('Build') {
            steps {
                echo 'Building from source..'
                // only build here; tests come later
                // use -B for batch mode to avoid interactive prompts
                withCredentials([usernamePassword(credentialsId: env.JFROG_CREDS,
                                                  usernameVariable: 'JFROG_USER',
                                                  passwordVariable: 'JFROG_PASSWORD')]) {
                    sh './mvnw -B clean package -DskipTests'
                }
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests..'
                withCredentials([usernamePassword(credentialsId: env.JFROG_CREDS,
                                                  usernameVariable: 'JFROG_USER',
                                                  passwordVariable: 'JFROG_PASSWORD')]) {
                    sh './mvnw -B test'
                }
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging as a Docker image..'
                // pin the build platform to linux/amd64 so that the resulting image is portable
                withCredentials([usernamePassword(credentialsId: env.JFROG_CREDS,
                                                  usernameVariable: 'JFROG_USER',
                                                  passwordVariable: 'JFROG_PASSWORD')]) {
                    sh '''
                        DOCKER_BUILDKIT=1 docker build ${DOCKER_NET} --platform linux/amd64 --provenance=false --sbom=false --build-arg JFROG_URL=${JFROG_URL} --build-arg JFROG_REPO=${JFROG_REPO} --secret id=jfrog_user,env=JFROG_USER --secret id=jfrog_password,env=JFROG_PASSWORD -t spring-petclinic:${BUILD_NUMBER} -t spring-petclinic:latest .
                    '''
                }

                // Prove the image is actually runnable, not just buildable.
                sh '''
                    docker rm -f petclinic-smoke || true
                    docker run -d --name petclinic-smoke -p 18080:8080 spring-petclinic:${BUILD_NUMBER}
                    for i in $(seq 1 60); do
                        if curl -sf http://localhost:18080/actuator/health > /dev/null; then
                            echo "Container healthy after ${i}s"
                            exit 0
                        fi
                        sleep 1
                    done
                    echo "Container failed to become healthy"
                    docker logs petclinic-smoke
                    exit 1
                '''

                // export the image as a tarball so it can be archived and downloaded from Jenkins
                sh 'docker save spring-petclinic:${BUILD_NUMBER} spring-petclinic:latest > spring-petclinic-${BUILD_NUMBER}.tar'
                archiveArtifacts artifacts: 'spring-petclinic-*.tar', fingerprint: true
            }
            post {
                always {
                    sh 'docker rm -f petclinic-smoke || true'
                    sh 'rm -f spring-petclinic-*.tar'
                }
            }
        }
    }
}