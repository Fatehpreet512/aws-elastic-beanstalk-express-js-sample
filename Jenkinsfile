pipeline {
  agent any

  environment {
    DOCKER_HOST = 'tcp://dind:2375'
    IMAGE_TAG   = "fatehpreet/eb-express:${env.BUILD_NUMBER}"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {
    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }

    stage('Install Dependencies') {
      steps {
        sh '''
          echo "📦 Installing dependencies..."
          docker run --rm -v "$PWD":/app -w /app node:16 npm install --production
        '''
      }
    }

    stage('Run Tests') {
      steps {
        sh '''
          echo "🧪 Running tests..."
          docker run --rm -v "$PWD":/app -w /app node:16 npm test
        '''
      }
    }

    stage('Security Scan (Snyk)') {
      steps {
        withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
          sh '''
            echo "🔒 Running Snyk Security Scan..."
            docker run --platform=linux/amd64 --rm \
              -e SNYK_TOKEN="$SNYK_TOKEN" \
              -v "$PWD":/app -w /app snyk/snyk:node snyk test --severity-threshold=high --json-file-output=snyk-results.json
          '''
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'snyk-results.json', onlyIfSuccessful: false
        }
      }
    }

    stage('Docker Build & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'REG_USER', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "🐳 Building Docker image..."
            docker build -t "$IMAGE_TAG" .

            echo "🔑 Logging into DockerHub..."
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            echo "📤 Pushing image to DockerHub..."
            docker push "$IMAGE_TAG"
          '''
        }
      }
    }
  }
}
