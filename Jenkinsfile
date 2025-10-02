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
      steps { checkout scm }
    }

    stage('Install Dependencies') {
      steps {
        sh '''
          docker run --rm \
            -v $(pwd):/app \
            -w /app node:16 \
            npm install --production
        '''
      }
    }

    stage('Run Tests') {
      steps {
        sh '''
          docker run --rm \
            -v $(pwd):/app \
            -w /app node:16 \
            npm test
        '''
      }
    }

    stage('Security Scan (Snyk)') {
      steps {
        withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
          sh '''
            docker run --rm \
              -e SNYK_TOKEN="$SNYK_TOKEN" \
              -v $(pwd):/app \
              -w /app snyk/snyk:stable bash -lc "
                snyk auth \$SNYK_TOKEN &&
                snyk test --severity-threshold=high --json-file-output=snyk-results.json
              "
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
        withCredentials([usernamePassword(credentialsId: 'REG_CREDS', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
          sh '''
            echo "$PASS" | docker login -u "$USER" --password-stdin
            docker build -t "$IMAGE_TAG" $(pwd)
            docker push "$IMAGE_TAG"
          '''
        }
      }
    }
  }
}
