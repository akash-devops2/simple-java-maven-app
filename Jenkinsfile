
// Jenkinsfile

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
        MAX_BUILDS_TO_KEEP = 5 // Keep the 5 latest SonarQube project analyses
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
                        # Ensure the artifact exists and then move it to the tagged-artifacts directory
                        # assuming 'my-app-${BUILD_NUMBER}.jar' is the output of mvn package
                        if [ -f "target/my-app-${BUILD_NUMBER}.jar" ]; then
                            mv target/my-app-${BUILD_NUMBER}.jar tagged-artifacts/${finalArtifact}
                        else
                            echo "Error: Expected artifact target/my-app-${BUILD_NUMBER}.jar not found!"
                            exit 1
                        fi
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
            // Optional: Add a 'always' post-stage action to remove the created Dockerfile if you want to keep the workspace clean
            post {
                always {
                    script {
                        sh "rm Dockerfile"
                    }
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
                    // Use a regex-safe prefix for project names derived from job name
                    def jobNameCleaned = env.JOB_NAME.replaceAll('[^a-zA-Z0-9-]', '-') // Replace non-alphanumeric with hyphen
                    def projectPrefix = "${jobNameCleaned}-"

                    echo "Attempting to delete old SonarQube projects for prefix: ${projectPrefix}..."

                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        def json = ''
                        try {
                            // Fetch all projects matching the prefix, up to 500
                            json = sh(script: """
                                curl -s -H "Authorization: Bearer ${SONAR_TOKEN}" \\
                                "${SONAR_URL}/api/projects/search?ps=500&q=${projectPrefix}"
                            """, returnStdout: true).trim()
                        } catch (e) {
                            echo "❌ Error fetching projects from SonarQube API: ${e.getMessage()}"
                            // You might want to fail the build here or just return depending on criticality
                            return
                        }

                        if (!json || json.trim().isEmpty() || json == "{}") {
                            echo "⚠️ No JSON data or empty JSON received from SonarQube API. Skipping deletion."
                            return
                        }

                        def parsed = [:]
                        try {
                            parsed = readJSON text: json
                        } catch (e) {
                            echo "❌ Failed to parse JSON response from SonarQube API: ${e.getMessage()}"
                            echo "Received JSON: ${json}" // Print malformed JSON for debugging
                            return // Exit if JSON is invalid
                        }

                        // Ensure 'components' array exists and is not null or empty
                        if (!parsed.components || parsed.components.isEmpty()) {
                            echo "No 'components' found in SonarQube API response for prefix '${projectPrefix}'. No projects to manage."
                            return
                        }

                        // Filter and sort projects:
                        // 1. Filter projects that start with our prefix and have a numerical suffix (like BUILD_NUMBER)
                        // 2. Sort them in ascending order of build number (oldest first)
                        def matchedProjects = parsed.components.findAll {
                            it.key.startsWith(projectPrefix) &&
                            it.key.replace(projectPrefix, '').isInteger() // Ensure the suffix is an integer
                        }.sort { a, b ->
                            def aNum = a.key.replace(projectPrefix, '').toInteger()
                            def bNum = b.key.replace(projectPrefix, '').toInteger()
                            aNum <=> bNum // Ascending sort for older builds
                        }

                        def totalProjects = matchedProjects.size()
                        echo "Found ${totalProjects} matching SonarQube projects with numerical suffixes for prefix '${projectPrefix}'."

                        if (totalProjects > MAX_BUILDS_TO_KEEP.toInteger()) {
                            // Drop the latest 'MAX_BUILDS_TO_KEEP' projects, leaving the oldest ones to delete
                            def toDelete = matchedProjects.drop(totalProjects - MAX_BUILDS_TO_KEEP.toInteger())

                            echo "Will delete ${toDelete.size()} old SonarQube projects."
                            toDelete.each { proj ->
                                echo "🗑️ Deleting old Sonar project: ${proj.key}"
                                def deleteResp = ''
                                try {
                                    // curl -s: silent, -w: write out, --upload-file: not needed for POST, -X: method
                                    deleteResp = sh(script: """
                                        curl -s -w "\\nHTTP_STATUS_CODE:%{http_code}" \\
                                        -H "Authorization: Bearer ${SONAR_TOKEN}" \\
                                        -X POST "${SONAR_URL}/api/projects/delete?project=${proj.key}"
                                    """, returnStdout: true).trim()

                                    def parts = deleteResp.split('HTTP_STATUS_CODE:')
                                    def status = parts.length > 1 ? parts[1].trim() : "unknown"
                                    if (status.startsWith("2")) { // 2xx status codes indicate success
                                        echo "✅ Successfully deleted ${proj.key} (HTTP status: ${status})"
                                    } else {
                                        echo "❌ Failed to delete ${proj.key} (HTTP status: ${status}). Response body (if any): ${parts[0]}"
                                    }
                                } catch (e) {
                                    echo "❌ Error during deletion of ${proj.key}: ${e.getMessage()}"
                                }
                            }
                        } else {
                            echo "No old SonarQube projects to delete. Keeping ${totalProjects} (max ${MAX_BUILDS_TO_KEEP})."
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
            echo 'Pipeline finished.'
        }
        success {
            echo 'Pipeline completed successfully! 🎉'
        }
        failure {
            echo 'Pipeline failed! 🔴 Check logs for details.'
        }
    }
}
