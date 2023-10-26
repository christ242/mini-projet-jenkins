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
