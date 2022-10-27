#!groovy

pipeline {
    // agent { label 'cbc-aws-g3-agent' }
    // agent { label 'gce-node' }
    // agent any
    // agent { label 'maven' }
    agent { label 'docker-agent' }

    environment {
        SERVICES_REPO_URL = 'git@github.com:CBQA-Org-Demo/vulnado.git'
        SERVICES_BRANCH = 'master'
        MODULE = 'vulnado'
        DOCKER_FILE = 'Dockerfile.vulnado'
        DOCKERHUB_ACNT = 'prakashsethuraman'
        GIT_CREDENTIAL_NAME = 'github-ssh-key'
    }

    options {
        // disableConcurrentBuilds()
        skipDefaultCheckout()
        skipStagesAfterUnstable()
        // timestamps()
        ansiColor('xterm')
    }

    stages {

        stage('Start') {
            steps {
              cleanWs()
              script {
                currentBuild.description = "${env.GIT_BRANCH} ${env.GIT_COMMIT}"
              }
            }
          }

        stage('Checkout') {
            steps {
                deleteDir()
                git branch: "${env.SERVICES_BRANCH}",
                    credentialsId: "${env.GIT_CREDENTIAL_NAME}",
                    url: "${env.SERVICES_REPO_URL}"
            }
        }

        stage('Pre-build') {
            steps {
              script {
                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "dockerhub_creds", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                //   GIT_SHORT_HASH = env.GIT_COMMIT.take(7)
                  TARGET_DOCKERHUB = sh (script: "echo ${DOCKERHUB_ACNT}/${MODULE}:latest", returnStdout: true).trim()     
                  sh '''
                  echo "Login into hub.docker.com"
                  docker login --username $USERNAME --password $PASSWORD
                  '''
                }
              }
            }
          }

        // stage('Build') {
        //     steps {
        //         withMaven(
        //             options: [junitPublisher(disabled: true, healthScaleFactor: 1.0)],
        //             publisherStrategy: 'EXPLICIT') {
        //                 sh 'mvn -Dmaven.test.failure.ignore clean package'
        //         }
        //     }
        // }

        stage('Docker-Build') {
            steps {
                script {
                    sh '''
                        ls -lrt *
                        docker build -t ${MODULE}  -f $DOCKER_FILE .
                        ID=$(docker create ${MODULE})
                        echo ${ID}
                        docker cp ${ID}:/src/target ./
                        docker rm ${ID}
                        pwd
                        ls target
                        '''
                }
            }
        }


        stage('SonarQube') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withMaven(
                        options: [junitPublisher(disabled: true, healthScaleFactor: 1.0)],
                        publisherStrategy: 'EXPLICIT') {
                            sh 'mvn sonar:sonar'
                    }
                }
            }
        }


        stage('Pipeline Compliance Check') {
            steps{
                script {
                final def (String response, String code) =
                    sh(script: """curl -X POST -d '{"requestSource": "CBCI", "requestId" : "1234123412341234", "requestTimestamp" : "2022-10-05T19:30:00Z", "details" : {"project" : "<<Project Name>>", "release" : "<<Release Name>>", "pipeline" : "<<Pipeline Name>>" } }' -s -w "\\n%{response_code}" https://www.qa.cbc.beescloud.com/api/external/webhook/pipeline-compliance-check""", returnStdout: true)                
                        .trim()
                        .tokenize('\n')

                if (code == null) {
                    code = Integer.parseInt(response)
                }

                if (code == "200") {
                    def json = readJSON text: response
                    def status = json.complianceCheckStatus
                    echo "status = $status"
                    if (status != "APPROVED") {
                        error "Pipeline compliance check - failed"
                    }
                } else {
                    echo "Failed to check compliance with CBC"
                    error "Failed to check compliance with CBC"
                }
                }
            }
        }    

        stage('DockerHub-Push') {
            steps {
            script {
                sh "docker tag $MODULE $TARGET_DOCKERHUB"
                sh "docker push -q $TARGET_DOCKERHUB"
                }
            }
        }

      

    }
}