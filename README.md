

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
                sh 'docker build -t ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG .'
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
           expression { GIT_BRANCH == 'origin/main' }
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

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/a05ca33c-4b4a-43f7-ab18-0f217eeedd1b)


We checked the github hook trigger in order to allow jenkins going to check the github containing the source code as soon as there's a modification . The pipeline will launched itsef as soon we commit something .

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/1869557f-c467-40bf-aa94-9a9db2def176)

On the image above , the specify to jenkins we are managing pipeline project and have provided the repo source code for git . No need to specify the credentials because we are working on a public project. We need the project to be runned on main branch .

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/670f12d8-49f8-41a3-9011-664fff92cbcb)


# Configuring the slack library 

![alt text]([image)![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/27894be4-0235-4b97-b8c7-d7b72d4a6460)

# Create secret for dockerhub
![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/0c364872-f9a4-46ed-8af1-ab8f96e66a74)

# Installing blueocean plugin
We install blue ocean plugin in order to have a better view of the pipeline scheme.
![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/7cf879a1-9ced-4698-8871-05cb5b96b7db)

# Viewing the jenkins tool with blue ocean tool

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/391c757a-97e7-4256-85a2-d5bb3fbfb2f1)

# Installing slack Notifier plugin
We first install the slack notification plugin.
![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/60dbebb7-898f-4585-be3c-ee5a00becc62)

Then , we are going to configure it in order to get notification from slack

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/98291b00-14d4-4903-aeb9-b26cc67de660)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/cd58f961-2044-4abe-9d0d-58df9e8da4cf)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/7c5b2dcf-b123-4204-85ea-3bf41725b40f)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/53315ac0-1a9f-4162-b850-6284600ebaf5)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/98eac707-e87f-40e9-a2cd-4ac6239870c4)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/ad1df781-2436-401c-81bc-507d58ef881d)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/6fa1eaf0-417c-49d1-ae2e-b2888b8eff09)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/0652226c-be79-4cb9-a962-a084bc3407d9)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/ba32ada7-6ac3-4270-b65c-39e7d9999e48)



# Deploying the Eazylab API 
We are going to deploy the eazylab machine for the deployment .

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/f33510fd-0bf4-4efb-a027-25e40ad612b7)

We can watch the docker container run for the eazylab API like this :

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/1dff5fee-af5d-4b9d-9627-6c2c1923d9aa)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/f1061c62-da15-4633-8f3d-e71dde03987b)


# Launching the pipeline CICD
After implementing everything , we are now going to launch the pipeline job .

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/00595d1c-49ee-4d73-b9f4-e533f39cfda7)

![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/e771079f-f897-4c74-ba7d-6a896716c227)

The job has been successful according to what we wrote in the jenksfile :
![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/b0c9845f-ec5d-41ef-9bf7-0bd369f55897)

# Watching the created container in the Easytraining API
 bash'''
[vagrant@jenkins mini-projet-jenkins]$ docker ps -a
CONTAINER ID   IMAGE                           COMMAND                  CREATED          STATUS          PORTS
                                                                                             NAMES
c1b7d144880f   kitepoye/staticwebsite:latest   "/docker-entrypoint.…"   58 seconds ago   Up 57 seconds   0.0.0.0:8090->80/tcp, :::8090->80/tcp
                                                                                             preprod-kitepoye
f2b39afb2655   eazytraining/eazylabs:latest    "python3 src/main.py"    4 hours ago      Up 4 hours      0.0.0.0:1993->1993/tcp, :::1993->1993/tcp
                                                                                             eazylabs
3571b257f10e   eazytraining/jenkins            "/bin/sh -c '/usr/sb…"   2 days ago       Up 2 days       22/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 80/tcp, 0.0.0.0:50000->50000/tcp, :::50000->50000/tcp, 0.0.0.0:443->8443/tcp, :::443->8443/tcp   jenkins-jenkins-1
 '''
 ![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/287c9e98-4208-4e6f-a0b3-f9c99acac6f1)


 # Watching the successful message on slack

 ![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/9c9666ae-99ea-41ff-9b7b-7ccb50d0e887)

 # Watchin the application on prod and staging environment 

 ![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/cb2eaf25-f259-4f33-8df3-50f59394d317)


![alt text]([image]![image](https://github.com/christ242/mini-projet-jenkins/assets/60726494/05c18fb1-0ccc-47b5-9c9b-eccd8a478eb4)

# Watching the pushed image on dockerhub








