pipeline {
    agent any

    environment {
        JAVA_HOME = "/usr/lib/jvm/java-21-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:/opt/maven/bin:$PATH"
        GIT_REPO_URL = 'https://github.com/akash-devops2/simple-java-maven-app.git'
        SONAR_URL = 'http://3.108.250.202:30900'
        SONAR_CRED_ID = 'sonar-token-id'
        NEXUS_URL = 'http://3.108.250.202:30001/repository/sample-releases/'
        NEXUS_DOCKER_REPO = '3.108.250.202:30002'
        NEXUS_CRED_ID = 'nexus-creds'
        NEXUS_DOCKER_CRED_ID = 'nexus-docker-creds'
        MAX_BUILDS_TO_KEEP = 5
        MVN_CMD = '/opt/maven/bin/mvn'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: "${GIT_REPO_URL}", branch: 'main'
            }
        }

        stage('Create Sonar Project') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        sh """
                            curl -s -o /dev/null -w "%{http_code}" -X POST \
                            -H "Authorization: Bearer ${SONAR_TOKEN}" \
                            "${SONAR_URL}/api/projects/create?project=${projectName}&name=${projectName}" || true
                        """
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        withSonarQubeEnv('MySonar') {
                            sh """
                                ${MVN_CMD} clean verify sonar:sonar \
                                -Dsonar.projectKey=${projectName} \
                                -Dsonar.host.url=${SONAR_URL} \
                                -Dsonar.token=${SONAR_TOKEN}
                            """
                        }
                    }
                }
            }
        }

        stage('Build and Tag Artifact') {
            steps {
                script {
                    def artifactName = "${env.JOB_NAME}-${env.BUILD_NUMBER}.jar".replace('/', '-')
                    sh "${MVN_CMD} clean package"
                    sh """
                        mkdir -p tagged-artifacts
                        cp target/*.jar tagged-artifacts/${artifactName}
                        echo "✅ Tagged JAR: ${artifactName}"
                    """
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                script {
                    def version = "1.0.${env.BUILD_NUMBER}"
                    def artifactId = "${env.JOB_NAME}".replace('/', '-')
                    def groupPath = "com/mycompany/app"
                    def finalArtifact = "${artifactId}-${version}.jar"
                    def uploadPath = "${groupPath}/${artifactId}/${version}"

                    sh "mv tagged-artifacts/${artifactId}-${env.BUILD_NUMBER}.jar tagged-artifacts/${finalArtifact}"

                    withCredentials([usernamePassword(credentialsId: "${NEXUS_CRED_ID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            curl -u $USERNAME:$PASSWORD \
                            --upload-file tagged-artifacts/${finalArtifact} \
                            ${NEXUS_URL}${uploadPath}/${finalArtifact}
                        """
                    }
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    def version = "1.0.${env.BUILD_NUMBER}"
                    def artifactName = "${env.JOB_NAME}-${version}.jar".replace('/', '-')
                    writeFile file: 'Dockerfile', text: """
                        FROM openjdk:21-jdk-slim
                        WORKDIR /app
                        COPY tagged-artifacts/${artifactName} app.jar
                        EXPOSE 8080
                        ENTRYPOINT ["java", "-jar", "app.jar"]
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def safeTag = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${safeTag}"
                    sh "docker build -t ${imageTag} ."
                }
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                script {
                    def safeTag = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${safeTag}"
                    withCredentials([usernamePassword(credentialsId: "${NEXUS_DOCKER_CRED_ID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            echo $PASSWORD | docker login ${NEXUS_DOCKER_REPO} -u $USERNAME --password-stdin
                            docker push ${imageTag}
                            docker logout ${NEXUS_DOCKER_REPO}
                        """
                    }
                }
            }
        }

        stage('Delete Old Sonar Projects') {
            steps {
                script {
                    def currentBuild = env.BUILD_NUMBER.toInteger()
                    def minBuildToKeep = currentBuild - MAX_BUILDS_TO_KEEP.toInteger()

                    if (minBuildToKeep > 0) {
                        withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                            for (int i = 1; i <= minBuildToKeep; i++) {
                                def oldProject = "${env.JOB_NAME}-${i}".replace('/', '-')
                                echo "🧹 Deleting old Sonar project: ${oldProject}"
                                sh """
                                    curl -s -o /dev/null -u $SONAR_TOKEN: -X POST \
                                    "${SONAR_URL}/api/projects/delete" \
                                    -d "project=${oldProject}" || true
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
