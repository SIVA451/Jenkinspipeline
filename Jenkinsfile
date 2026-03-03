// Jenkinsfile (Declarative Pipeline) - Sample

pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
    timeout(time: 45, unit: 'MINUTES')
  }

  parameters {
    choice(name: 'ENV', choices: ['dev', 'test', 'prod'], description: 'Target environment')
    booleanParam(name: 'RUN_SONAR', defaultValue: true, description: 'Run SonarQube scan + quality gate')
    booleanParam(name: 'BUILD_DOCKER', defaultValue: true, description: 'Build & push Docker image')
  }

  environment {
    // Change these as per your setup
    APP_NAME       = 'sample-app'
    REGISTRY       = '123456789012.dkr.ecr.eu-west-1.amazonaws.com'
    IMAGE_REPO     = "${REGISTRY}/${APP_NAME}"
    IMAGE_TAG      = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

    // If using SonarQube, configure Jenkins "Configure System" -> SonarQube servers with this name
    SONAR_SERVER   = 'sonarqube-server'
    SONAR_PROJECT  = 'sample-app'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD'
      }
    }

    stage('Build') {
      steps {
        sh '''
          set -e
          echo "Building..."
          # Example: Node
          if [ -f package.json ]; then
            npm ci
            npm run build
          # Example: Maven
          elif [ -f pom.xml ]; then
            mvn -B -DskipTests clean package
          # Example: Python
          elif [ -f requirements.txt ]; then
            python -m venv .venv
            . .venv/bin/activate
            pip install -U pip
            pip install -r requirements.txt
          else
            echo "No recognized build file found. Add your build commands."
          fi
        '''
      }
    }

    stage('Test') {
      steps {
        sh '''
          set -e
          echo "Testing..."
          # Example: Node
          if [ -f package.json ]; then
            npm test || true
          # Example: Maven
          elif [ -f pom.xml ]; then
            mvn -B test
          # Example: Python (pytest)
          elif [ -d .venv ] || [ -f requirements.txt ]; then
            [ -d .venv ] && . .venv/bin/activate || true
            pytest -q || true
          else
            echo "No tests configured."
          fi
        '''
      }
    }

    stage('SonarQube Scan') {
      when { expression { return params.RUN_SONAR } }
      steps {
        withSonarQubeEnv("${SONAR_SERVER}") {
          sh '''
            set -e
            echo "Running SonarQube scan..."
            # Example generic scanner; adjust to your language/build tool
            sonar-scanner \
              -Dsonar.projectKey=${SONAR_PROJECT} \
              -Dsonar.projectName=${SONAR_PROJECT} \
              -Dsonar.sources=. \
              -Dsonar.host.url=$SONAR_HOST_URL \
              -Dsonar.login=$SONAR_AUTH_TOKEN
          '''
        }
      }
    }

    stage('Quality Gate') {
      when { expression { return params.RUN_SONAR } }
      steps {
        // Requires "SonarQube Scanner for Jenkins" plugin and webhook configured in SonarQube
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Docker Build & Push') {
      when { expression { return params.BUILD_DOCKER } }
      steps {
        sh '''
          set -e
          echo "Docker build & push..."
          docker version

          # If you are using ECR, ensure agent has AWS CLI + permissions
          aws --version
          aws ecr get-login-password --region eu-west-1 \
            | docker login --username AWS --password-stdin ${REGISTRY}

          docker build -t ${IMAGE_REPO}:${IMAGE_TAG} .
          docker push ${IMAGE_REPO}:${IMAGE_TAG}

          echo "Pushed: ${IMAGE_REPO}:${IMAGE_TAG}"
        '''
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          set -e
          echo "Deploying to ${ENV}..."
          # Replace with your real deployment:
          # - kubectl apply / helm upgrade
          # - ssh + systemctl restart
          # - argo app sync
          echo "Example: kubectl -n ${ENV} set image deploy/${APP_NAME} ${APP_NAME}=${IMAGE_REPO}:${IMAGE_TAG}"
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
    failure {
      echo "❌ Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
    always {
      // Optional cleanup
      sh 'docker system prune -f || true'
      cleanWs(deleteDirs: true, notFailBuild: true)
    }
  }
}
