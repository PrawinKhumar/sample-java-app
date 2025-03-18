pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'SonarQubeServer'  // Name from Jenkins > Manage Jenkins > Configure Global Tool
        SONARQUBE_TOKEN = credentials('sonarqube-token') // Add this in Jenkins Credentials
        DOCKERHUB_CREDS = credentials('dockerhub-creds') // Add this in Jenkins Credentials
        DOCKER_IMAGE = "prawinkhumar/sample-java-app:latest"
    }

    stages {
        stage('Checkout Code') {
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
                sh '''
                    mkdir -p dependency-check-report
                    docker run --rm \
                        -v $(pwd):/src \
                        owasp/dependency-check \
                        --project sample-java-app \
                        --scan /src \
                        --format HTML \
                        --out /src/dependency-check-report
                '''
            }
        }

        stage('SAST - SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=sample-java-app -Dsonar.login=$SONARQUBE_TOKEN'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $DOCKER_IMAGE ."
            }
        }

        stage('Scan Docker Image - Trivy') {
            steps {
                sh "trivy image $DOCKER_IMAGE || true"
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push $DOCKER_IMAGE
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    docker run -d -p 8080:8080 $DOCKER_IMAGE
                    sleep 10
                    curl -f http://localhost:8080 || echo "Smoke test failed"
                '''
            }
        }
    }
}
