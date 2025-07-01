pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        SONAR_TOKEN = credentials('sonar-token')
        SONAR_HOST = 'http://65.2.137.238:30900'
        MAVEN_HOME = tool 'Maven 3'
        NEXUS_MAVEN_URL = 'http://65.2.137.238:30801/repository/sample-releases/'
        NEXUS_DOCKER_URL = '65.2.137.238:30002'
        IMAGE_NAME = 'my-app'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/akash-devops2/sample-java-sonar-nexus.git', branch: 'main'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                        ${MAVEN_HOME}/bin/mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=assignment-sonar-project \
                        -Dsonar.host.url=${SONAR_HOST} \
                        -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Maven Build') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn clean package"
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                script {
                    def artifactVersion = "${BUILD_NUMBER}"
                    def finalArtifact = "my-app-${artifactVersion}.jar"
                    sh """
                        cp target/*.jar tagged-artifacts/${finalArtifact}
                        cp target/*.jar tagged-artifacts/my-app-1.0.${artifactVersion}.jar

                        curl -v -u admin:admin123 --upload-file tagged-artifacts/${finalArtifact} ${NEXUS_MAVEN_URL}${finalArtifact}
                        curl -v -u admin:admin123 --upload-file tagged-artifacts/my-app-1.0.${artifactVersion}.jar ${NEXUS_MAVEN_URL}my-app-1.0.${artifactVersion}.jar
                    """
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    def versionTag = "${BUILD_NUMBER}"
                    sh """
                        docker build -t ${NEXUS_DOCKER_URL}/${IMAGE_NAME}:${versionTag} .
                        docker push ${NEXUS_DOCKER_URL}/${IMAGE_NAME}:${versionTag}
                    """
                }
            }
        }

        stage('Cleanup old SonarQube reports') {
            steps {
                script {
                    def projectKeyPrefix = "assignment-sonar-project"

                    def response = sh(
                        script: """
                            curl -s -H "Authorization: Bearer ${SONAR_TOKEN}" \
                            "${SONAR_HOST}/api/components/search?qualifiers=TRK"
                        """,
                        returnStdout: true
                    ).trim()

                    def json = readJSON text: response
                    def projects = json.components.findAll { it.key.startsWith(projectKeyPrefix) }

                    // Simulate sort by analysis date if available (Sonar doesn't return full analysis info here)
                    projects.sort { it.name }

                    println "Matched Sonar Projects: ${projects*.key}"
                    println "Total Projects Found: ${projects.size()}"

                    if (projects.size() > 5) {
                        def oldProjects = projects[0..<(projects.size() - 5)]
                        oldProjects.each { proj ->
                            sh """
                                curl -X POST -H "Authorization: Bearer ${SONAR_TOKEN}" \
                                "${SONAR_HOST}/api/projects/delete?project=${proj.key}"
                            """
                            println "Deleted Sonar project: ${proj.key}"
                        }
                    } else {
                        println "Less than or equal to 5 projects; skipping deletion."
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
