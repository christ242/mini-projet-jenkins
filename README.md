# mini-projet-jenkins
This project deals with the work I did as a student at devops's bootcam from eazytraining .
# Objectives
The aim of this project is to set up a CICD pipeline through jenkins tools . In order to achieve this project , we first need get the source code , dockerzise it and build a pipeline CICD for the built image .

# Installing and Implementing Jenkins
The jenkins is installed from a vagrant machine I deployed before .

![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/f72915f5-e054-4c3f-937d-1871fa8818d9)


Now I'm going to implement the jenkins servers and access it

![alt text]([image](![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/2a6ba969-dffd-45e1-8d7e-b90f0991b676)

Now , we are going to install the plugins .


![alt text]([image](![image](![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/60b9353d-de15-4fc9-af9e-cfd711a69652)


![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/0662be24-a2d7-4c77-914b-2e06a3effbc0)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/0f7dc5c1-353d-4100-bf72-05b6a7ce0cf3)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/883fd0c5-e400-44ae-b60c-3e1ab24881cb)


# Writing the Dockerfile
We are going to dockerzise the source code .
```bash
FROM nginx:1.21.1
LABEL maintainer="Christ BAGAMBOULA"
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y curl && \
    apt-get install -y git
RUN rm -Rf /usr/share/nginx/html/*
RUN git clone https://github.com/diranetafen/static-website-example.git /usr/share/nginx/html
CMD nginx -g 'daemon off;'
````
# Jenksfile
This how the jenkinsfile I have writen looks like:
```bash
  /* import shared library */
@Library('kitepoye-slack-share-library')_

pipeline {
    environment {
        IMAGE_NAME = "staticwebsite"
        APP_EXPOSED_PORT = "80"
        IMAGE_TAG = "latest"
        STAGING = "kitepoye-staging"
        PRODUCTION = "kitepoye-prod"
        DOCKERHUB_ID = "kitepoye"
        DOCKERHUB_PASSWORD = credentials('dockerhub_password')
        APP_NAME = "kitepoye"
        STG_API_ENDPOINT = "172.28.128.140:1993"
        STG_APP_ENDPOINT = "172.28.128.140:${PORT_EXPOSED}90"
        PROD_API_ENDPOINT = "172.28.128.140:1993"
        PROD_APP_ENDPOINT = "172.28.128.140:${PORT_EXPOSED}"
        INTERNAL_PORT = "80"
        EXTERNAL_PORT = "${PORT_EXPOSED}"
        CONTAINER_IMAGE = "${DOCKERHUB_ID}/${IMAGE_NAME}:${IMAGE_TAG}"
    }
    agent none
    stages {
       stage('Build image') {
           agent any
           steps {
              script {
                sh 'docker build -t ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG ./mini-projet-jenkins/'
              }
           }
       }
       stage('Run container based on builded image') {
          agent any
          steps {
            script {
              sh '''
                  echo "Cleaning existing container if exist"
                  docker ps -a | grep -i $IMAGE_NAME && docker rm -f $IMAGE_NAME
                  docker run --name $IMAGE_NAME -d -p $APP_EXPOSED_PORT:$INTERNAL_PORT  ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG
                  sleep 5
              '''
             }
          }
       }
       stage('Test image') {
           agent any
           steps {
              script {
                sh '''
                   curl -v 172.17.0.1:$APP_EXPOSED_PORT | grep -i "Dimension"
                '''
              }
           }
       }
       stage('Clean container') {
          agent any
          steps {
             script {
               sh '''
                   docker stop $IMAGE_NAME
                   docker rm $IMAGE_NAME
               '''
             }
          }
      }

      stage ('Login and Push Image on docker hub') {
          agent any
          steps {
             script {
               sh '''
                   echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_ID --password-stdin
                   docker push ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG
               '''
             }
          }
      }

      stage('STAGING - Deploy app') {
      agent any
      steps {
          script {
            sh """
              echo  {\\"your_name\\":\\"${APP_NAME}\\",\\"container_image\\":\\"${CONTAINER_IMAGE}\\", \\"external_port\\":\\"${EXTERNAL_PL_PORT}90\\", \\"internal_port\\":\\"${INTERNAL_PORT}\\"}  > data.json
              curl -k -v -X POST http://${STG_API_ENDPOINT}/staging -H 'Content-Type: application/json'  --data-binary @data.json  2>&1 |  | grep 200
            """
          }
        }

     }
     stage('PROD - Deploy app') {
       when {
           expression { GIT_BRANCH == 'origin/master' }
       }
     agent any

       steps {
          script {
            sh """
              echo  {\\"your_name\\":\\"${APP_NAME}\\",\\"container_image\\":\\"${CONTAINER_IMAGE}\\", \\"external_port\\":\\"${EXTERNAL_PL_PORT}\\", \\"internal_port\\":\\"${INTERNAL_PORT}\\"}  > data.json
              curl -k -v -X POST http://${PROD_API_ENDPOINT}/prod -H 'Content-Type: application/json'  --data-binary @data.json  2>&1 | gr grep 200
            """
          }
       }
     }
  }
  post {
       success {
         slackSend (color: '#00FF00', message: "CHRIST- SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL}) - PROD OD URL => http://${PROD_APP_ENDPOINT} , STAGING URL => http://${STG_APP_ENDPOINT}")
         }
      failure {
            slackSend (color: '#FF0000', message: "CHRIST - FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")      
          }
    }
}
````
 ## Configuration of Jenkins Tools

 We first are going to create a pipeline project named "mini-projet-jenkins" as shown below 
 ![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/4afb0422-bce9-48e2-98a7-90f4cbbcde23)

We specify the name of the github project and the paramters for deployment already specified in our jenksfile.

 ![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/e6b5a72e-a6fb-44e6-89d5-6c09b3d25226)

 The github project name has be specified too , like this :
 ![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/f337f45c-537b-49eb-9630-56e3386c4c95)


