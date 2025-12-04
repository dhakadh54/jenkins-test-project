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

    //If using DinD sidecar, you may use env.DOCKER_HOST to point docker client to that daemon.
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

    stage('1 - Pull image from env.REGISTRY') {
      steps {
        // run docker pull in dind sidecar container
        // note: the agent must have a 'dind' container in the pod template OR be a node with docker
        container('dind') {
          sh docker pull ${env.REGISTRY} 
        }
      }
    }

    stage('2 - Generate manifest at runtime') {
      steps {
        // Use jnlp container to write manifest into workspace
        container('jnlp') {
          sh """
            set -eux
            cat > ${env.MANIFEST_FILE} <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${env.IMAGE_NAME}-deploy
  env.NAMESPACE: ${env.NAMESPACE}
  labels:
    app: ${env.IMAGE_NAME}-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${env.IMAGE_NAME}-deploy
  template:
    metadata:
      labels:
        app: ${env.IMAGE_NAME}-deploy
    spec:
      containers:
        - name: ${env.IMAGE_NAME}
          image: ${env.REGISTRY}
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ${env.IMAGE_NAME}-svc
  env.NAMESPACE: ${env.NAMESPACE}
spec:
  selector:
    app: ${env.IMAGE_NAME}-deploy
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
            echo "Wrote manifest ${env.MANIFEST_FILE}:"
            sed -n '1,200p' ${env.MANIFEST_FILE} || true
          """
        }
      }
    }

    stage('3 - kubectl apply') {
      steps {
        container('jnlp') {
          withCredentials([file(credentialsId: env.env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -eux
              export PATH="$HOME/bin:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"

              // ensure env.NAMESPACE exists
              kubectl get env.NAMESPACE ${env.NAMESPACE} >/dev/null 2>&1 || kubectl create env.NAMESPACE ${env.NAMESPACE}

              // apply the manifest from workspace
              kubectl apply -n ${env.NAMESPACE} -f ${WORKSPACE}/${env.MANIFEST_FILE}
            '''
          }
        }
      }
    }

    stage('4 - Rollout verification') {
      steps {
        container('jnlp') {
          withCredentials([file(credentialsId: env.env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -eux
              export PATH="$HOME/bin:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"

              kubectl rollout status deployment/${env.IMAGE_NAME}-deploy -n ${env.NAMESPACE} --timeout=120s
            '''
          }
        }
      }
    }

    stage('5 - Pod validation using kubectl get pods') {
      steps {
        container('jnlp') {
          withCredentials([file(credentialsId: env.env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -eux
              export PATH="$HOME/bin:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"
              echo "Pods in env.NAMESPACE ${env.NAMESPACE}:"
              kubectl get pods -n ${env.NAMESPACE} -o wide
            '''
          }
        }
      }
    }

    stage('6 - Run a basic post-deploy test using curl') {
      steps {
        // Use jnlp container and kubectl to run an ephemeral curl pod inside the cluster
        container('jnlp') {
          withCredentials([file(credentialsId: env.env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -eux
              export PATH="$HOME/bin:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"
              PODNAME="curl-test-$(date +%s)"

              kubectl run "${PODNAME}" --rm -i --restart=Never --image=curlimages/curl -n ${env.NAMESPACE} --command -- sh -c '
                set -eux
                for i in 1 2 3 4 5; do
                  echo "Attempt ${i}: curl -sS -m 5 http://${env.IMAGE_NAME}-svc/"
                  if curl -sS -m 5 http://${env.IMAGE_NAME}-svc/; then
                    echo "Service responded"
                    exit 0
                  else
                    echo "Not ready yet; sleeping 2s"
                    sleep 2
                  fi
                done
                echo "Service did not respond" >&2
                exit 1
              '
            '''
          }
        }
      }
    }

    stage('7 - Cleanup using kubectl delete -f <manifest>') {
      steps {
        container('jnlp') {
          withCredentials([file(credentialsId: env.env.KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
            sh '''
              set -eux || true
              export PATH="$HOME/bin:$PATH"
              export KUBECONFIG="${KUBECONFIG_FILE}"
              kubectl delete -n ${env.NAMESPACE} -f ${WORKSPACE}/${env.MANIFEST_FILE} --ignore-not-found=true || true
              echo "Cleanup completed."
            '''
          }
        }
      }
    }

  } // stages

  post {
    always {
      archiveArtifacts artifacts: "${env.MANIFEST_FILE}", fingerprint: true, allowEmptyArchive: true
      echo 'Pipeline finished.'
    }
    failure {
      echo 'Pipeline failed — check console output.'
    }
  }
}
