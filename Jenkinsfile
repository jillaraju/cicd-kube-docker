pipeline {

    agent { label 'KOPS' }   // 👈 FIX: same node for all stages

    tools {
        maven "maven3"
    }

    environment {
        registry = "rajujilla/vproappdock"
        registryCredential = 'dockerhub'
    }

    stages {

        stage('Build & Test') {
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

        stage('Code Analysis - Checkstyle') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Code Analysis - SonarQube') {
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

        // 🔍 Debug (optional)
        stage('Check WAR File') {
            steps {
                sh '''
                echo "Workspace:"
                pwd
                echo "Listing target folder:"
                ls -l target/
                '''
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build("${registry}:V${BUILD_NUMBER}")
                }
            }
        }

        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V${BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Local Image') {
            steps {
                sh "docker rmi ${registry}:V${BUILD_NUMBER} || true"
            }
        }

        stage('Kubernetes Deploy') {
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
