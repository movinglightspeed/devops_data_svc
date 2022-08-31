pipeline {
    agent any 

    tools {nodejs "nodejs"}

    environment {
        registryCredential = 'dockerhub'
        imageName = 'stanleys/internalapp'
        dockerImage = ''
        PATH = "/usr/sbin:/usr/bin:/sbin:/bin:${env.PATH}"
        }
    stages {
        stage('Run the tests') {
             agent {
                docker { 
                    image 'node:14-alpine'
                    args '-e HOME=/tmp -e NPM_CONFIG_PREFIX=/tmp/.npm'
                    args "-u root"  
                    alwaysPull true
                    reuseNode true
                }
            }
            steps {
                echo "PATH is: ${env.PATH}"
                echo 'Retrieve source from github. run npm install and npm test' 
                git branch: 'main',
                    url: 'https://github.com/movinglightspeed/devops_data_svc.git'
                echo 'repo files'
                sh 'ls -a'
                echo 'install dependencies'
                sh 'npm install'
                echo 'Run tests'
                sh 'npm test'
                echo 'Testing completed'   
            }
        }
        stage('SonarQube analysis') {
      steps {
        script {
          def scannerHome = tool 'sonarqube';
          withSonarQubeEnv('sonarqube') {
            sh "./sonar-scanner-4.7.0.2747-linux/bin/sonar-scanner"
          }
         }
        }
       }
       stage("Quality gate") {
      steps {
        script {
          def qualitygate = waitForQualityGate()
          sleep(10)
          if (qualitygate.status != "OK") {
            waitForQualityGate abortPipeline: true
           }
          }
         }
        }
        stage('Building image') {
            steps{
                script {
                    echo 'build the image'
                    dockerImage = docker.build("${env.imageName}:${env.BUILD_ID}")
                    echo "${env.imageName}:${env.BUILD_ID}"
                    echo 'image built'
                }
            }
            }
        stage('Push Image') {
            steps{
                script {
                    echo 'push the image to docker hub'
                    docker.withRegistry('',registryCredential){
                        dockerImage.push("${env.BUILD_ID}")
                  }
                }
            }
        }     
         stage('deploy to k8s') {
             agent {
                docker { 
                    image 'ubuntu:latest'
                    args '-u root'          
                    reuseNode true
                    alwaysPull true
                        }
                    }
            steps {
                echo 'Get cluster credentials'
                sh 'pwd && whoami'
                sh 'apt-get update && apt-get upgrade && apt-get install -y gnupg && apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 871920D1991BC93C'
                sh 'apt --fix-broken install -y'
                sh 'apt-get install apt-transport-https ca-certificates gnupg -y'
                sh 'echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] http://packages.cloud.google.com/apt cloud-sdk main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key --keyring /usr/share/keyrings/cloud.google.gpg  add - && apt-get update -y && apt-get install google-cloud-cli -y'
                sh 'apt-get install google-cloud-sdk-gke-gcloud-auth-plugin -y'
                sh 'gcloud auth activate-service-account ${gaccount} --key-file=devopsbootcamp-355721-d2c37704b9c8.json'
                sh 'gcloud config set account ${gaccount}'
                sh 'gcloud container clusters get-credentials cluster-1 --zone us-central1-c --project devopsbootcamp-355721'
                sh "kubectl set image deployment/devops-data-svc data-svc-container=${env.imageName}:${env.BUILD_ID}"
              }
            }       
        stage('Remove local docker images') {
            steps{
                script {
                    echo 'push the image to docker hub' 
                }
                // sh "docker rmi $imageName:latest"
                sh "docker rmi $imageName:$BUILD_NUMBER"
            }
        }
    }
}
