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
                            curl -s -o /dev/null -w "%{http_code}" -X POST \\
                            -H "Authorization: Bearer ${SONAR_TOKEN}" \\
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
                                ${MVN_CMD} clean verify sonar:sonar \\
                                -Dsonar.projectKey=${projectName} \\
                                -Dsonar.host.url=${SONAR_URL} \\
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
                    def artifactName = "my-app-${BUILD_NUMBER}.jar"
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
                    def version = "1.0.${BUILD_NUMBER}"
                    def artifactId = "my-app"
                    def groupPath = "com/mycompany/app"
                    def finalArtifact = "${artifactId}-${version}.jar"
                    def uploadPath = "${groupPath}/${artifactId}/${version}"

                    sh """
                        mv tagged-artifacts/my-app-${BUILD_NUMBER}.jar tagged-artifacts/${finalArtifact}
                    """

                    withCredentials([usernamePassword(credentialsId: "${NEXUS_CRED_ID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            curl -u $USERNAME:$PASSWORD \\
                            --upload-file tagged-artifacts/${finalArtifact} \\
                            ${NEXUS_URL}${uploadPath}/${finalArtifact}
                        """
                    }
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    writeFile file: 'Dockerfile', text: """
                        FROM openjdk:21-jdk-slim
                        WORKDIR /app
                        COPY tagged-artifacts/my-app-*.jar app.jar
                        EXPOSE 8080
                        ENTRYPOINT ["java", "-jar", "app.jar"]
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                }
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                script {
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${BUILD_NUMBER}"
                    withCredentials([usernamePassword(credentialsId: "${NEXUS_DOCKER_CRED_ID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            echo $PASSWORD | docker login http://${NEXUS_DOCKER_REPO} -u $USERNAME --password-stdin
                            docker push ${imageTag}
                            docker logout http://${NEXUS_DOCKER_REPO}
                        """
                    }
                }
            }
        }

        stage('Delete Old Sonar Projects') {
            steps {
                script {
                    def projectPrefix = "${env.JOB_NAME}-".replace('/', '-')

                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        def json = sh(script: """
                            curl -s -H "Authorization: Bearer ${SONAR_TOKEN}" \\
                            "${SONAR_URL}/api/projects/search?ps=500&q=${projectPrefix}"
                        """, returnStdout: true).trim()

                        if (!json || json == "{}") {
                            echo "⚠️ No JSON data received from SonarQube API."
                            return
                        }

                        def parsed = readJSON text: json
                        def matchedProjects = parsed.components.findAll {
                            it.key.startsWith(projectPrefix) &&
                            it.key.replace(projectPrefix, '').isInteger()
                        }.sort { a, b ->
                            def aNum = a.key.replace(projectPrefix, '').toInteger()
                            def bNum = b.key.replace(projectPrefix, '').toInteger()
                            bNum <=> aNum
                        }

                        def totalProjects = matchedProjects.size()
                        echo "Found ${totalProjects} matching SonarQube projects."

                        if (totalProjects > MAX_BUILDS_TO_KEEP.toInteger()) {
                            def toDelete = matchedProjects.drop(MAX_BUILDS_TO_KEEP.toInteger())
                            toDelete.each { proj ->
                                echo "🗑️ Deleting old Sonar project: ${proj.key}"
                                def deleteResp = sh(script: """
                                    curl -s -w "\\nHTTP_STATUS_CODE:%{http_code}" \\
                                    -H "Authorization: Bearer ${SONAR_TOKEN}" \\
                                    -X POST "${SONAR_URL}/api/projects/delete?project=${proj.key}"
                                """, returnStdout: true).trim()

                                def parts = deleteResp.split('HTTP_STATUS_CODE:')
                                def status = parts.length > 1 ? parts[1].trim() : "unknown"
                                echo "🔁 Deleted ${proj.key} with HTTP status: ${status}"
                            }
                        } else {
                            echo "✅ No old SonarQube projects to delete. (${totalProjects} found)"
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
