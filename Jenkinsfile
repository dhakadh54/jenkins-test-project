// Jenkinsfile - Declarative pipeline integrated from your scripted snippet
pipeline {
  agent { label 'test-jenkins-cluster' }

  environment {
    REGISTRY        = "nginx:alpine"                     // image to pull
    IMAGE_NAME      = "nginx"
    IMAGE_TAG       = "alpine"
    MANIFEST_FILE   = "deployment.yaml"
    KUBECONFIG_CRED = "kubeconfig-file-test-cluster"    // Jenkins Secret File ID
    NAMESPACE       = "test-ns"

    //If using DinD sidecar, you may use DOCKER_HOST to point docker client to that daemon.
    DOCKER_HOST     = "tcp://dind:2375"
    DOCKER_TLS_VERIFY = "0"
   // KUBECTL_DOWNLOAD_URL = "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  }

  options {
    timeout(time: 30, unit: 'MINUTES')
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }
  }

stage('Install kubectl using jnlp') {
  steps {
    container('jnlp') {
      sh '''
        set -eux

        mkdir -p "$HOME/bin"

        # Compute latest stable kubectl URL dynamically
        RELEASE=$(curl -L -s https://dl.k8s.io/release/stable.txt)
        URL="https://dl.k8s.io/release/${RELEASE}/bin/linux/amd64/kubectl"

        echo "Downloading kubectl from: $URL"

        curl -LO "$URL"
        chmod +x kubectl
        mv kubectl "$HOME/bin/"

        export PATH="$HOME/bin:$PATH"
        kubectl version --client
      '''
    }
  }
}

stage('1 - Pull image from registry') {
  steps {
    container('dind') {
      sh '''
        set -eux || true
        # ensure DOCKER_HOST is available for docker client
        echo "DOCKER_HOST=${DOCKER_HOST}"
        if command -v docker >/dev/null 2>&1; then
          docker pull ${REGISTRY} || true
        else
          echo "docker CLI not present in this container; cluster will pull image on pod creation"
        fi
      '''
    }
  }
}

    stage('2 - Generate manifest at runtime') {
      steps {
        // Use jnlp container to write manifest into workspace
        container('jnlp') {
          sh """
            set -eux
            cat > ${MANIFEST_FILE} <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${IMAGE_NAME}-deploy
  namespace: ${NAMESPACE}
  labels:
    app: ${IMAGE_NAME}-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${IMAGE_NAME}-deploy
  template:
    metadata:
      labels:
        app: ${IMAGE_NAME}-deploy
    spec:
      containers:
        - name: ${IMAGE_NAME}
          image: ${REGISTRY}
          ports:
            - containerPort: 80
          """
        }
      }
    }

    stage('3 - kubectl apply') {
      steps {
        container('jnlp') {
          withCredentials([file(credentialsId: KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -eux
              export PATH="$HOME/bin:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"

              # ensure NAMESPACE exists
              #kubectl get NAMESPACE ${NAMESPACE} >/dev/null 2>&1 || kubectl create NAMESPACE ${NAMESPACE}

              # apply the manifest from workspace
              kubectl apply -n ${NAMESPACE} -f ${WORKSPACE}/${MANIFEST_FILE}
            '''
          }
        }
      }
    }

    stage('4 - Rollout verification') {
      steps {
        container('jnlp') {
          withCredentials([file(credentialsId: KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -eux
              export PATH="$HOME/bin:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"

              kubectl rollout status deployment/${IMAGE_NAME}-deploy -n ${NAMESPACE} --timeout=120s
            '''
          }
        }
      }
    }

    stage('5 - Pod validation using kubectl get pods') {
      steps {
        container('jnlp') {
          withCredentials([file(credentialsId: KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -eux
              export PATH="$HOME/bin:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"
              echo "Pods in NAMESPACE ${NAMESPACE}:"
              kubectl get pods -n ${NAMESPACE} -o wide
            '''
          }
        }
      }
    }

    stage('6 - Cleanup using kubectl delete -f <manifest>') {
      steps {
        container('jnlp') {
          withCredentials([file(credentialsId: KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -eux || true
              export PATH="$HOME/bin:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"
              kubectl delete -n ${NAMESPACE} -f ${WORKSPACE}/${MANIFEST_FILE} --ignore-not-found=true || true
              echo "Cleanup completed."
            '''
          }
        }
      }
    }

  } // stages

  post {
    always {
      archiveArtifacts artifacts: "${MANIFEST_FILE}", fingerprint: true, allowEmptyArchive: true
      echo 'Pipeline finished.'
    }
    failure {
      echo 'Pipeline failed — check console output.'
    }
  }
}
