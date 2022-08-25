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
                script {
                    System.setProperty("org.jenkinsci.plugins.durabletask.BourneShellScript.LAUNCH_DIAGNOSTICS=true"); 
                }
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
                    image 'google/cloud-sdk:latest'
                    args '-e HOME=/tmp'
                    reuseNode true
                        }
                    }
            steps {
                echo 'Get cluster credentials'
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
