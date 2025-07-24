pipeline {
  agent any

  environment {
    IMAGE_TAG = "v.${BUILD_ID}.0"
    DOCKER_HUB_USERNAME = credentials("DOCKER_HUB_USER")
    DOCKER_HUB_PASSWORD = credentials("DOCKER_HUB_PASS")
    KUBE_CONFIG = credentials("config") 
  }

  stages {
     stage('test') {
       agent any
       steps {
         sh '''
           sudo apt-get update
           sudo apt-get install -y python3 python3-pip
           python3 -m pip install --upgrade pip setuptools wheel
           pip install fastapi pytest httpx
         '''
       }
     }

    stage('Docker build') {
      agent any
      steps {
        script {
            sh """
              docker build -t $DOCKER_HUB_USERNAME/gateway:$IMAGE_TAG -f gateway/Dockerfile gateway

              docker build -t $DOCKER_HUB_USERNAME/users:$IMAGE_TAG -f users/Dockerfile users

              docker build -t $DOCKER_HUB_USERNAME/orders:$IMAGE_TAG -f orders/Dockerfile orders

              sleep 5
            """
          
        }
      }
    }

    stage(' Docker run') { // run container from the built images
      agent any
      steps {
        script {
            sh """
              set -e
              for service in gateway users orders; do
                docker rm -f \$service || true
              done

              docker run -d --name gateway -p 8000:8000 $DOCKER_HUB_USERNAME/gateway:$IMAGE_TAG
              docker run -d --name users -p 8001:8000 $DOCKER_HUB_USERNAME/users:$IMAGE_TAG
              docker run -d --name orders -p 8002:8000 $DOCKER_HUB_USERNAME/orders:$IMAGE_TAG
            """
          
        }
      }
    }

    stage('Docker push') {
      agent any
      steps {
        script {
            sh """
              docker login -u $DOCKER_HUB_USERNAME -p $DOCKER_HUB_PASSWORD
              docker push $DOCKER_HUB_USERNAME/gateway:$IMAGE_TAG
              docker push $DOCKER_HUB_USERNAME/users:$IMAGE_TAG
              docker push $DOCKER_HUB_USERNAME/orders:$IMAGE_TAG
            """
          
        }
      }
    }

    stage('Deploy') {
      agent any
      steps {
        script {
          sh '''
          rm -Rf .kube
          mkdir -p .kube
          ls
          cat "$KUBE_CONFIG" > .kube/config
          chmod 600 .kube/config
            '''
          def environments = ['dev', 'qa', 'staging']
          for (envName in environments) {
            deployTo(envName)
          }

          input message: 'Deploy to PROD?'
          deployTo('prod')
        }
      }
    }
  }
}

def deployTo(envName) {
  sh """
    # Gateway
    if helm status gateway --kubeconfig ${KUBE_CONFIG} --namespace ${envName} >/dev/null 2>&1; then
      helm uninstall gateway --kubeconfig ${KUBE_CONFIG} --namespace ${envName}
    fi

    helm install gateway ./fastapi-app/gateway \
      --kubeconfig ${KUBE_CONFIG} \
      --namespace ${envName} \
      --create-namespace \
      --set image.repository=$DOCKER_HUB_USERNAME/gateway \
      --set image.tag=$IMAGE_TAG \
      --set gateway.env.USERS_SERVICE_URL=http://users-service:8000 \
      --set gateway.env.ORDERS_SERVICE_URL=http://orders-service:8000
    
    # Users
    if helm status users --kubeconfig ${KUBE_CONFIG} --namespace ${envName} >/dev/null 2>&1; then
      helm uninstall users --kubeconfig ${KUBE_CONFIG} --namespace ${envName}
    fi

    helm install users ./fastapi-app/users \
      --kubeconfig ${KUBE_CONFIG} \
      --namespace ${envName} \
      --create-namespace \
      --set image.repository=$DOCKER_HUB_USERNAME/users \
      --set image.tag=$IMAGE_TAG
    
    # orders
    if helm status orders --kubeconfig ${KUBE_CONFIG} --namespace ${envName} >/dev/null 2>&1; then
      helm uninstall orders --kubeconfig ${KUBE_CONFIG} --namespace ${envName}
    fi

    helm upgrade --install orders ./fastapi-app/orders \
      --force \
      --kubeconfig ${KUBE_CONFIG} \
      --namespace ${envName} \
      --create-namespace \
      --set image.repository=$DOCKER_HUB_USERNAME/orders \
      --set image.tag=$IMAGE_TAG
  """
}
