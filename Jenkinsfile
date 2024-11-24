pipeline {
    agent any

    tools {
        maven "MAVEN3.9"
    }

    environment {
        registry = "kvait/vproappdock"
        registryCredential = 'dockerhub'
    }

    stages {
        // Stage to build the application
        stage('BUILD') {
            steps {
                // Clean and install the application, skipping tests
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    // Archive the built WAR files
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        // Stage to run unit tests
        stage('UNIT TEST') {
            steps {
                // Execute unit tests
                sh 'mvn test'
            }
        }

        // Uncomment the following stages if needed

        // Stage for integration testing
        /*
        stage('INTEGRATION TEST') {
            steps {
                // Verify the application, skipping unit tests
                sh 'mvn verify -DskipUnitTests'
            }
        }
        */

        // Stage for code analysis with Checkstyle
        /*
        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                // Run Checkstyle analysis
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        */

        // Stage to build the Docker image
        stage('Building image') {
            steps {
                script {
                    // Build the Docker image with the specified tag
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                }
            }
        }

        // Stage to deploy the Docker image
        stage('Deploy Image') {
            steps {
                script {
                    // Push the Docker image to the registry
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        // Stage to remove unused Docker images
        stage('Remove Unused Docker Image') {
            steps {
                // Remove the Docker image with the specific build number
                sh "docker rmi $registry:$BUILD_NUMBER"
            }
        }

        // Stage for code analysis with SonarQube
        stage('CODE ANALYSIS with SONARQUBE') {
            environment {
                scannerHome = tool 'mysonarscanner4'
            }
            steps {
                withSonarQubeEnv('sonar-pro') {
                    // Run SonarQube scanner with specified parameters
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                       -Dsonar.projectName=vprofile-repo \
                       -Dsonar.projectVersion=1.0 \
                       -Dsonar.sources=src/ \
                       -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                       -Dsonar.junit.reportsPath=target/surefire-reports/ \
                       -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                       -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                // Wait for the quality gate to pass or fail
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // Stage for deploying to Kubernetes
        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                // Deploy the application using Helm
                sh "helm upgrade --install --force vproifle-stack helm/vprofilecharts --set appimage=${registry}:${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}

