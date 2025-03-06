def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
]

pipeline {
    agent any

    tools {
        maven 'MAVEN3.9'
        jdk 'JDK17'
    }

    stages {
        stage('Fetch code') {
            steps {
                echo 'Fetching code..'
                git branch: 'atom', url: 'https://github.com/hkhcoder/vprofile-project.git'
            }
        }

        stage('Unit Test') {
            steps {
                    echo 'Testing..'
                    sh 'mvn test'
            }
        }

        stage('Build') {
            steps {
                echo 'Building..'
                sh 'mvn install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving the Artifacts..'
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                echo 'Checkstyle Analysis..'
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Sonar Code Analysis') {
            environment {
                scannerHome = tool 'sonar6.2'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh '''${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportsPath=target/checkstyle-result.xml'''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Upload to Nexus') {
            environment {
                NEXUS_VERSION = "nexus3"
                NEXUS_PROTOCOL = "http"
                NEXUS_URL = "172.31.18.160:8081"
                NEXUS_REPO = "vprofile-repo"
                NEXUS_CREDENTIAL_ID = "nexuslogin"
            }
            steps {
                echo 'Uploading to Nexus..'
                nexusArtifactUploader(
                    nexusVersion: "${NEXUS_VERSION}",
                    protocol: "${NEXUS_PROTOCOL}",
                    nexusUrl: "${NEXUS_URL}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: "${NEXUS_REPO}",
                    credentialsId: "${NEXUS_CREDENTIAL_ID}",
                    artifacts: [
                        [artifactId: 'vproapp', classifier: '', file: 'target/vprofile-v2.war', type: 'war']
                    ]
                )
            }
        }
    }

    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#ci-cd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}"
        }
    }
}