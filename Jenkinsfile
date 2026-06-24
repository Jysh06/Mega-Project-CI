pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    environment {
        SCANNER_HOME = tool('sonar-scanner')
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {

        stage('GIT CHECKOUT') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Jysh06/Mega-Project-CI.git'
            }
        }

        stage('COMPILE') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('TESTING') {
            steps {
                sh 'mvn test'
            }
        }

        stage('TRIVY FILE SCAN') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }

        stage('SONAR ANALYSIS') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=gcbank \
                        -Dsonar.projectName=gcbank \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('QUALITY CHECK') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('BUILD') {
            steps {
                sh 'mvn package'
            }
        }

        stage('NEXUS DEPLOYMENT') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'gconfig',
                    jdk: 'jdk17',
                    maven: 'maven3',
                    traceability: true
                ) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('DOCKER BUILD AND TAG') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t jysh06/bankapp:${IMAGE_TAG} ."
                    }
                }
            }
        }

        stage('SCAN IMAGE') {
            steps {
                sh "trivy image --format table -o image-report.html jysh06/bankapp:${IMAGE_TAG}"
            }
        }

        stage('DOCKER PUSH') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push jysh06/bankapp:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('UPDATE MANIFEST IN MEGA CD') {
            steps {
                script {
                    cleanWs()

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'gitcred',
                            usernameVariable: 'GIT_USERNAME',
                            passwordVariable: 'GIT_TOKEN'
                        )
                    ]) {

                        sh '''
                            git clone https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/Jysh06/Mega-Project-CD.git

                            cd Mega-Project-CD

                            sed -i "s|jysh06/bankapp:.*|jysh06/bankapp:${IMAGE_TAG}|g" Manifest/manifest.yaml

                            echo "Updated manifest file contents:"
                            cat Manifest/manifest.yaml

                            git config user.name "Jenkins"
                            git config user.email "jenkins@example.com"

                            git add Manifest/manifest.yaml

                            git commit -m "Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"

                            git push origin main
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                        <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                            <h2>${jobName} - Build ${buildNumber}</h2>

                            <div style="background-color: ${bannerColor}; padding: 10px;">
                                <h3 style="color: white;">
                                    Pipeline Status: ${pipelineStatus.toUpperCase()}
                                </h3>
                            </div>

                            <p>
                                Check the <a href="${BUILD_URL}">console output</a>.
                            </p>
                        </div>
                    </body>
                    </html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'jayeshpat23.jp@gmail.com',
                    from: 'jayeshpat23.jp@gmail.com',
                    replyTo: 'jayeshpat23.jp@gmail.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}

