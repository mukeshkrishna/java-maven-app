#!/usr/bin/env groovy

library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
    [$class: 'GitSCMSource',
     remote: 'https://github.com/mukeshkrishna/Jenkins_Shared_Library.git',
     credentialsId: 'gitlab-credentials'
    ]
)

pipeline {
    agent any
    tools {
        maven 'maven-3.8'
    }
}
    stages {
		stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    echo "MATCHER -- ${matcher}"
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "mukeshdevops/java-maven-app:$version-$BUILD_NUMBER"
                }
            }
        }

        stage('build app') {
            steps {
               script {
                  echo 'building application jar...'
                  buildJar()
               }
            }
        }
        stage('build image') {
            steps {
                script {
                   echo 'building docker image...'
                   buildImage(env.IMAGE_NAME)
                   dockerLogin()
                   dockerPush(env.IMAGE_NAME)
                }
            }
        }
        stage('provision server') {
			
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
				// we can over-write default variable defined in terraform file by setting environment varibles TF_VAR_<variable name defined in variables.tf file>
                TF_VAR_env_prefix = 'test'
            }
            steps {
                script {
					// dir() is used execute the commands in the block at a particular directory
                    dir('terraform') {
                        sh "terraform init"
                        sh "terraform apply --auto-approve"
                        
						// Capture the output defined in main.tf file to get the public_ip dynamically after the server is provisioned and initialized
						// we can store it in environment variable and can be used in other stages
						EC2_PUBLIC_IP = sh(
                            script: "terraform output ec2_public_ip",
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }
        stage('deploy') {
			
            environment {
                DOCKER_CREDS = credentials('docker-hub-repo')
            }
            steps {
                script {
                   echo "waiting for EC2 server to initialize" 
				   // force jenkins to wait till the server is initialised after provisioning
                   sleep(time: 90, unit: "SECONDS") 

                   echo 'deploying docker image to EC2...'
                   echo "${EC2_PUBLIC_IP}"
					
				   // we need to configure docker login in EC-2 server so we are referencing the DOCKER_CREDS stored in jenkins credential store
				   // When defining DOCKER_CREDS as environment variable, we can access password and username using DOCKER_CREDS_USR,DOCKER_CREDS_PSW like done below	
                   def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
                   def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"
				   
				   echo "Transforming docker-compose.yaml file with latest tag"
                   sh "sed -i s/tag/${IMAGE_NAME}/ docker-compose.yaml"

                   sshagent(['server-ssh-key']) {
                       sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
                       sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
                       sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
                   }
                }
            }
        }
		stage('commit version update') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'gitlab-creds', passwordVariable: 'GITPASS', usernameVariable: 'GITUSER')]) {
                        // git config here for the first time run
                        echo "Few information"
                        sh 'git status'
                        sh 'git branch'
                        sh 'git config --list'
                        echo "Information over"

                        sh 'git config --global user.email "jenkins@example.com"'
                        sh 'git config --global user.name "jenkins"'
                        sh "git remote set-url origin https://${GITUSER}:${GITPASS}@gitlab.com/${GITUSER}/java-maven-app.git"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh "git push origin HEAD:jenkins-jobs"
                    }
                }
            }
        }
    }
}
