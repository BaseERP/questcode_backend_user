def LABEL_ID = "questcode-${UUID.randomUUID().toString()}"


podTemplate(
    name: 'questcode', 
    namespace: 'devops', 
    label: LABEL_ID, 
    containers:[
        //livenessProbe: containerLivenessProbe(execArgs: '', failureThreshold: 0, initialDelaySeconds: 0, periodSeconds: 0, successThreshold: 0, timeoutSeconds: 0), 

        containerTemplate(args: 'cat', command: '/bin/sh -c', image: 'docker', name: 'docker-container', ttyEnabled: true),
        containerTemplate(args: 'cat', command: '/bin/sh -c', image: 'lachlanevenson/k8s-helm:v2.11.0',  name: 'helm-container',  ttyEnabled: true)
    ], 
    volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')] 
)

{
    def REPOS
    def IMAGE_NAME = "backend-user"
    def KUBE_NAMESPACE
    def IMAGE_VERSION 
    def IMAGE_POSFIX = ""
    def ENVIRONMENT 
    def GIT_BRANCH 
    def CHARTMUSEUM_URL = "http://helm-chartmuseum:8080"
    def HELM_CHART_NAME = "questcode/backend-user"
    def HELM_DEPLOY_NAME  
    //def NODE_PORT = "30022"
    def INGRESS_HOST = "questcode.org"

    //Start Pipeline
    node(LABEL_ID) {
        stage('Checkout') {
            echo 'Iniciando Clone do repositorio'
            REPOS = checkout scm
            GIT_BRANCH = REPOS.GIT_BRANCH
            //com base no branch, direciona ao ambiente correto
            if(GIT_BRANCH.equals("master")) {
               KUBE_NAMESPACE = "prod"
               ENVIRONMENT = "production"
            } else if(GIT_BRANCH.equals("develop")) {
               KUBE_NAMESPACE = "staging"
               ENVIRONMENT = "staging"
               IMAGE_POSFIX = "-RC"
               //NODE_PORT = "30020"
               INGRESS_HOST = "staging.questcode.org"
            } else {
                def error =  "Não existe pipeline para a branch ${GIT_BRANCH}"
                echo error
                throw new Exception(error)
            }
            HELM_DEPLOY_NAME = KUBE_NAMESPACE + "-backend-user" 
            IMAGE_VERSION = sh label: '', returnStdout: true, script: 'sh read-package-version.sh'
            IMAGE_VERSION =IMAGE_VERSION.trim()+IMAGE_POSFIX
        }
        stage('Package') {
            container('docker-container') {
               echo 'Iniciando o empacotamento com docker'
               withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER')]) {
                    sh "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}"
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION} ."
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION}"
               }

            }
        } 
        
        stage('Deploy') {
            //CLIENTE DO HELM
            container('helm-container') {
                echo 'Iniciando Deploy Helm'
                sh 'helm init --client-only'
                sh "helm repo add questcode ${CHARTMUSEUM_URL}"
                sh 'helm repo update'
                try{
                    //Fazer helm upgrade
                    sh "helm upgrade --namespace=${KUBE_NAMESPACE} ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set image.tag=${IMAGE_VERSION} --set ingress.hosts[0].host=${INGRESS_HOST}"
                } catch(Exception e) {
                    //Fazer helm install
                    sh "helm install --namespace=${KUBE_NAMESPACE} --name ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set image.tag=${IMAGE_VERSION} --set ingress.hosts[0].host=${INGRESS_HOST}"
                }
            }

        }
    }
}