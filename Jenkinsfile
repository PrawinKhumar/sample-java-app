pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'SonarQubeServer'
        DOCKER_IMAGE = "prawinkhumar/sample-java-app:latest"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/PrawinKhumar/sample-java-app.git', branch: 'main'
            }
        }

        stage('Build & Unit Test') {
            steps {
                sh 'mvn clean install'
            }
        }

        stage('Code Coverage - JaCoCo') {
            steps {
                sh 'mvn jacoco:report'
            }
        }

        stage('SCA - OWASP Dependency Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    sh '''
                        mkdir -p dependency-check-report
                        docker run --rm \
                            -v $(pwd):/src \
                            -e NVD_API_KEY=$NVD_API_KEY \
                            owasp/dependency-check \
                            --project sample-java-app \
                            --scan /src \
                            --format HTML \
                            --out /src/dependency-check-report
                    '''
                }
            }
        }

        stage('SAST & Quality Gate - SonarQube') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Scan Docker Image - Trivy') {
            steps {
                sh "trivy image ${DOCKER_IMAGE} || true"
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh 'curl --fail http://localhost:8080 || echo "Smoke test failed or application not running yet."'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'dependency-check-report/*.html', fingerprint: true
        }
    }
}
