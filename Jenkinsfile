pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner-7.3.0.5189' // Jenkins Sonar Scanner name
    }

    stages {
        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/amitlinuxgit/amazon-Devsecops.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            $SCANNER_HOME/bin/sonar-scanner \
                                -Dsonar.projectName=amazon \
                                -Dsonar.projectKey=amazon \
                                -Dsonar.sources=. \
                                -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 3, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                    }
                }
            }
        }

        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }

        stage("OWASP FS Scan") {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan ./ 
                    --disableYarnAudit 
                    --disableNodeAudit 
                ''',
                odcInstallation: 'dp-check'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Build Docker Image") {
            steps {
                script {
                    env.IMAGE_TAG = "amitsawant31/amazon:${BUILD_NUMBER}"

                    sh "docker rmi -f amazon ${env.IMAGE_TAG} || true"
                    sh "docker build -t amazon ."
                }
            }
        }

        stage("Tag & Push to DockerHub") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker-cred', variable: 'dockerpwd')]) {
                        sh "docker login -u amitsawant31 -p ${dockerpwd}"
                        sh "docker tag amazon ${env.IMAGE_TAG}"
                        sh "docker push ${env.IMAGE_TAG}"

                        sh "docker tag amazon amitsawant31/amazon:latest"
                        sh "docker push amitsawant31/amazon:latest"
                    }
                }
            }
        }

        stage("Trivy Scan Image") {
            steps {
                script {
                    sh """
                        echo '🔍 Running Trivy scan on ${env.IMAGE_TAG}'
                        trivy image -f json -o trivy-image.json ${env.IMAGE_TAG}
                        trivy image -f table -o trivy-image.txt ${env.IMAGE_TAG}
                    """
                }
            }
        }

        stage("Deploy to Container") {
            steps {
                script {
                    sh "docker rm -f amazon || true"
                    sh "docker run -d --name amazon -p 80:80 ${env.IMAGE_TAG}"
                }
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'GitHub User'

                emailext(
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is an automated Amazon CI/CD pipeline report.</p>
                        <p><b>Project:</b> ${env.JOB_NAME}</p>
                        <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                        <p><b>Status:</b> ${buildStatus}</p>
                        <p><b>Triggered by:</b> ${buildUser}</p>
                        <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    to: 'nikhildevaws25@gmail.com',
                    from: 'nikhildevaws25@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivyfs.txt,trivy-image.json,trivy-image.txt,dependency-check-report.xml'
                )

                // Clean workspace at the end only
            cleanWs()
            }
        }
    }
}
