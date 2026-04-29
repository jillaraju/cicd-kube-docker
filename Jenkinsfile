pipeline {

    agent none   // 👈 control execution per stage

    tools {
        maven "maven3"
    }

    environment {
        registry = "rajujilla/vproappdock"
        registryCredential = 'dockerhub'
    }

    stages {

        // 🔹 Build + Test (optimized)
        stage('Build & Test') {
            agent any
            steps {
                sh 'mvn clean verify'
            }
            post {
                success {
                    echo 'Archiving WAR file...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        // 🔹 Code Analysis
        stage('Code Analysis - Checkstyle') {
            agent any
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Code Analysis - SonarQube') {
            agent any
            environment {
                scannerHome = tool 'mysonarscanner4'
            }
            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh """
                    ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile-repo \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    """
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // 🔥 Debug (optional but useful)
        stage('Check WAR File') {
            agent { label 'KOPS' }
            steps {
                sh '''
                echo "Workspace:"
                pwd
                echo "Checking target folder:"
                ls -l target/
                '''
            }
        }

        // 🔹 Docker Build
        stage('Build App Image') {
            agent { label 'KOPS' }   // 👈 Docker installed here
            steps {
                script {
                    dockerImage = docker.build("${registry}:V${BUILD_NUMBER}")
                }
            }
        }

        // 🔹 Push Image
        stage('Upload Image') {
            agent { label 'KOPS' }
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V${BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        // 🔹 Cleanup
        stage('Remove Local Image') {
            agent { label 'KOPS' }
            steps {
                sh "docker rmi ${registry}:V${BUILD_NUMBER} || true"
            }
        }

        // 🔹 Deploy
        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                sh """
                helm upgrade --install vprofile-stack helm/vprofilecharts \
                --set appimage=${registry}:V${BUILD_NUMBER} \
                --namespace prod --create-namespace
                """
            }
        }
    }
}
