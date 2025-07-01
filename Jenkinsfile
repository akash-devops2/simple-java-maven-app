pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'MySonar' // SonarQube Jenkins configuration name
        MAVEN_HOME = tool 'Maven 3'
        NEXUS_REPO = 'sample-releases'
        NEXUS_URL = 'http://3.108.250.202:30001/repository/sample-releases'
        DOCKER_REGISTRY = '3.108.250.202:30002'
        DOCKER_IMAGE_NAME = 'my-app'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git 'https://github.com/akash-devops2/sample-java-sonar-nexus.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv("${SONARQUBE_SERVER}") {
                        sh """
                            ${MAVEN_HOME}/bin/mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=assignment-${BUILD_NUMBER} \
                            -Dsonar.host.url=http://3.108.250.202:30900 \
                            -Dsonar.token=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Build and Tag Artifact') {
            steps {
                script {
                    sh "${MAVEN_HOME}/bin/mvn clean package"
                    sh """
                        mkdir -p tagged-artifacts
                        cp target/my-app-1.0-SNAPSHOT.jar tagged-artifacts/assignment-${BUILD_NUMBER}.jar
                        echo ✅ Tagged JAR: assignment-${BUILD_NUMBER}.jar
                    """
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                script {
                    def finalArtifact = "assignment-1.0.${BUILD_NUMBER}.jar"
                    sh "mv tagged-artifacts/assignment-${BUILD_NUMBER}.jar tagged-artifacts/${finalArtifact}"
                    withCredentials([usernamePassword(credentialsId: 'nexus-user', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            curl -u $USERNAME:$PASSWORD --upload-file tagged-artifacts/${finalArtifact} \
                            ${NEXUS_URL}/com/mycompany/app/assignment/1.0.${BUILD_NUMBER}/${finalArtifact}
                        """
                    }
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    writeFile file: 'Dockerfile', text: """
                        FROM openjdk:17-jdk-slim
                        COPY tagged-artifacts/assignment-${BUILD_NUMBER}.jar /app.jar
                        ENTRYPOINT ["java", "-jar", "/app.jar"]
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def dockerTag = "${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:assignment-${BUILD_NUMBER}"
                    sh "docker build -t ${dockerTag} ."
                    echo "✅ Docker image built: ${dockerTag}"
                }
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-docker-creds', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    script {
                        def dockerTag = "${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:assignment-${BUILD_NUMBER}"
                        sh "echo $PASSWORD | docker login ${DOCKER_REGISTRY} -u $USERNAME --password-stdin"
                        sh "docker push ${dockerTag}"
                        echo "✅ Docker image pushed: ${dockerTag}"
                    }
                }
            }
        }

        stage('Delete Old Sonar Projects') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            curl -s -u admin:${SONAR_TOKEN} 'http://3.108.250.202:30900/api/projects/search' | \
                            jq -r '.components[] | select(.key | startswith("assignment-")) | .key' | \
                            sort -r | tail -n +6 | \
                            while read key; do
                                curl -X POST -u admin:${SONAR_TOKEN} \
                                "http://3.108.250.202:30900/api/projects/delete?project=\$key"
                            done
                        """
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
