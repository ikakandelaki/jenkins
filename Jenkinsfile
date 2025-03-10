pipeline {
    agent any

    tools {
        maven 'MAVEN3.9'
        jdk 'JDK17'
    }

    environment {
        REGISTRY_CREDENTIAL = 'ecr:eu-north-1:awscreds'
        IMAGE_NAME = '113580812368.dkr.ecr.eu-north-1.amazonaws.com/vprofileappimg'
        VPROFILE_REGISTRY = 'https://113580812368.dkr.ecr.eu-north-1.amazonaws.com'
        CLUSTER = "vprofile"
        SERVICE = "vprofileappsvc"
    }

    stages {
        stage('Fetch code') {
            steps {
                echo 'Fetching code..'
                git branch: 'docker', url: 'https://github.com/hkhcoder/vprofile-project.git'
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

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build(IMAGE_NAME + ":${BUILD_NUMBER}", './Docker-files/app/multistage/')
                }
            }
        }

        stage('Upload App Image') {
            steps {
                script {
                    docker.withRegistry(VPROFILE_REGISTRY, REGISTRY_CREDENTIAL) {
                        dockerImage.push("${BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy to ecs') {
            steps {
                withAWS(credentials: 'awscreds', region: 'eu-north-1') {
                    sh 'aws ecs update-service --cluster ${CLUSTER} --service ${SERVICE} --force-new-deployment'
                }
            }
        }

        stage('Remove Images') {
            steps {
                sh 'docker rmi -f $(docker images -a -q)'
            }
        }
    }
}