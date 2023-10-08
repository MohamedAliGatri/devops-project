#!/usr/bin/env groovy
pipeline{
    agent any
    tools{
        maven "M2_HOME"
    }
    environment {
        IMAGE_NAME = 'gatrimohamedali/devops-project'
    }
    stages{
        stage("FS trivy scan"){
            steps{
              script{
                sh "trivy fs ."
              }
            }
        }
        stage("Test stage"){
            steps{
              script{
                echo "Testing stage"
              }
            }
        }
        /*stage("SonarTest integration"){
            steps{
                withSonarQubeEnv(installationName: 'SonarQubeServer') {
                    sh "mvn sonar:sonar"
                }
            }
        }*/
        stage("Incrementing version"){
          steps{
            script {
              sh 'mvn build-helper:parse-version versions:set \
                -DnewVersion=" \\\${parsedVersion.majorVersion}.\\\${parsedVersion.nextMinorVersion}"\
                versions:commit'
              def matcher = readFile("pom.xml") =~'<version>(.+)</version>'
              def version = matcher[1][1]
              //env.APP_VERSION="$version-$BUILD_NUMBER" some ADDS BUILD NUMBER TO VERSION
              env.APP_VERSION="$version".trim()
            }
          }
        }
        stage("Maven Package"){
            steps{
              script{
                sh "mvn clean package"
              }
            }
        }
        /*stage('Push to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '192.168.0.5:8081',
                    groupId: 'com.esprit.examen',
                    version: "${APP_VERSION}",
                    repository: 'learning',
                    credentialsId: 'nexus_credentials',
                    artifacts: [
                      [
                        artifactId: 'achat',
                        classifier: '',
                        file: "target/achat-${APP_VERSION}.jar",
                        type: 'jar'
                      ]
                    ]
                  )
                
            }
          }*/
          stage("login & build docker"){
            steps {
              script{
                withCredentials([usernamePassword(credentialsId:'docker_credentials', passwordVariable:'DOCKER_PASS', usernameVariable:'DOCKER_USER')]){
                  sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                  sh "docker build -t ${IMAGE_NAME}:${APP_VERSION} ."
                }
              }
            }
          }
          stage("tag and push docekr image"){
            steps {
              script{
                sh "docker push ${IMAGE_NAME}:${APP_VERSION}"
              }
            }
          }
          /*stage("terraform provisioning"){
            environment{
              AWS_ACCESS_KEY_ID = credentials('aws_access_key')
              AWS_SECRET_ACCESS_KEY = credentials('aws_secret_key')
            }
            steps{
              script{
                dir('terraform'){
                  sh 'terraform init'
                  sh 'terraform apply --auto-approve'
                  env.EC2_PUBLIC_IP = sh(
                    script: "terraform output ec2_ip",
                    returnStdout: true
                  ).trim()
                }
              }
            }
          }*/
          stage("commit version increment - state file"){
            environment{
              GITHUB_ACCESS_KEY = credentials('github_access_key')
            }
            steps{
              script{
                withCredentials([usernamePassword(credentialsId:'github_credentials',passwordVariable:'GIT_PASS',usernameVariable:'GIT_USER')]){
                  sh "git remote set-url origin https://${GITHUB_ACCESS_KEY}@github.com/${GIT_USER}/devops-project.git"
                  sh "git add ."
                  sh 'git commit -m "jenkins: version bump - state file commit"'
                  sh 'git push origin HEAD:Di'
                }
              }
            }
          }
          stage("cleaning up"){
            steps{
              script{
                sh "docker image rm ${IMAGE_NAME}:${APP_VERSION}"
              }
            }
          }
          /*stage("deploy on ec2 server"){
            environment{
              DOCKER_CREDS = credentials('docker_credentials')
              FULL_IMAGE_NAME = "${IMAGE_NAME}:${APP_VERSION}"
            }
            steps {
              script{
                echo "waiting for the ec2 to initialize"
                sleep(time: 180, unit: "SECONDS")
                
                def shellCmd = "bash ./server-cmds.sh ${FULL_IMAGE_NAME} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
                def ec2Instance = "ubuntu@${EC2_PUBLIC_IP}"
                def homeDir = "/home/ubuntu"

                sshagent(['ssh_key_to_ec2']) {
                  sh "scp -o StrictHostKeyChecking=no -o ServerAliveInterval=300 server-cmds.sh ${ec2Instance}:${homeDir}"
                  sh "scp -o StrictHostKeyChecking=no -o ServerAliveInterval=300 docker-compose.yml ${ec2Instance}:${homeDir}"
                  sh "scp -o StrictHostKeyChecking=no -o ServerAliveInterval=300 prometheus.yml ${ec2Instance}:${homeDir}"
                  sh "ssh -o StrictHostKeyChecking=no -o ServerAliveInterval=300 ${ec2Instance} ${shellCmd}"
                }
              }
            }
          }*/

    }
    
}