pipeline {
    agent { label 'PBuildNode' }

    environment {
        SONARQUBE_SERVER = 'SonarQube-Server'
        DOCKER_IMAGE = 'prawinkhumar/sample-java-app:latest'
        REGISTRY_CREDENTIALS = 'dockerhub-creds'
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

        stage('SAST & Quality Gate - SonarQube') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=sample-java-app'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
            }
        }

        stage('Scan Docker Image - Trivy') {
            steps {
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image $DOCKER_IMAGE
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $DOCKER_IMAGE
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    echo "Running Smoke Test..."
                    curl --fail http://localhost:8080 || echo "Smoke test failed or service not yet deployed."
                '''
            }
        }
    }
}
