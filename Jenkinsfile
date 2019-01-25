//  Copyright 2018 IBM
//    Licensed under the Apache License, Version 2.0 (the "License");
//    you may not use this file except in compliance with the License.
//    You may obtain a copy of the License at
//        http://www.apache.org/licenses/LICENSE-2.0
//    Unless required by applicable law or agreed to in writing, software
//    distributed under the License is distributed on an "AS IS" BASIS,
//    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
//    See the License for the specific language governing permissions and
//    limitations under the License.

podTemplate(label: 'mypod', cloud: 'kubernetes', serviceAccount: 'default', namespace: 'default',
    volumes: [
        hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
],
    containers: [
        containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'docker' , image: 'docker:17.06.1-ce', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'node'   , image: 'node:8', ttyEnabled: true, comand: 'cat')
  ]) {

    node('mypod') {
        checkout scm
        container('node') {
            stage('test') {
                sh """
                #!/bin/bash
                cd nodeApp
                echo "Installing dependencies"
                npm install
                echo "Starting linting and unit testing"
                npm test
                """
            }
        }
        container('docker') {
            stage('Build Docker Image') {
                sh """
                #!/bin/bash
                ls
                cd nodeApp
                docker build -t ${env.DOCKER_USERNAME}/'jenkins-demo-app':${env.BUILD_NUMBER} .
                """
            }
            stage('Push Docker Image to Registry') {
                withCredentials([usernamePassword(credentialsId: 'REGISTRY_CREDENTIALS',
                                               usernameVariable: 'USERNAME',
                                               passwordVariable: 'PASSWORD')]) {
                    sh """
                    #!/bin/bash
                    docker login -u ${USERNAME} -p ${PASSWORD}
                    docker push ${env.DOCKER_USERNAME}/'jenkins-demo-app':${env.BUILD_NUMBER}
                    """
                }
            }
        }
        container('kubectl') {
            stage('Deploy new Docker Image') {
                sh """
                #!/bin/bash

                echo ${env.BRANCH_NAME}

                if [ ${env.BRANCH_NAME} == 'test' ]; then
                    DEPLOYMENT=`kubectl --namespace=default get deployments -l app="jenkins-demo-app-test" -o name`
                    echo 'In test branch. Deploying to test env'
                    kubectl --namespace=default set image \${DEPLOYMENT} jenkins-demo-app=${env.DOCKER_USERNAME}/jenkins-demo-app:${env.BUILD_NUMBER}

                elif [ ${env.BRANCH_NAME} == 'master' ]; then
                    DEPLOYMENT=`kubectl --namespace=default get deployments -l app=jenkins-demo-app -o name`
                    echo 'In master branch. Deploying to prod env'
                    kubectl --namespace=default set image \${DEPLOYMENT} jenkins-demo-app=${env.DOCKER_USERNAME}/jenkins-demo-app:${env.BUILD_NUMBER}
                else
                    echo "In unexpected branch $env.BRANCH_NAME. Exiting."
                    exit 1
                fi

                if [ \${?} -ne "0" ]; then
                    echo "No deployment to update"
                    echo "Creating deployment"
                    kubectl apply kube/nodeApp.yaml
                fi 


                # Update Deployment
                kubectl --namespace=default rollout status \${DEPLOYMENT}
                """
            }
        }
    }
}
