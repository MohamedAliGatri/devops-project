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
        stage('OWASP Dependency-Check Vulnerabilities') {
            steps {
                dependencyCheck additionalArguments: ''' 
                            -o './'
                            -s './'
                            -f 'ALL' 
                            --prettyPrint''', odcInstallation: 'OWASP Dependency-Check Vulnerabilities'
                
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }
        stage("Test stage"){
            steps{
              script{
                echo "Testing stage"
              }
            }
        }
        stage("SonarTest integration"){
            steps{
              container("maven:3.6.3-openjdk-11-slim"){
                withSonarQubeEnv(installationName: 'SonarQubeServer') {
                    sh "mvn sonar:sonar"
                }
              }
            }
        }
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
                def image = 'maven:3.6.3-openjdk-11-slim'
                def workspacePath = pwd() 
                sh "docker run -v $workspacePath:/output $image mvn clean package -Dmaven.test.skip=true -DskipTests"
              }
            }
        }
        stage('Push to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '192.168.0.2:8081',
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
          }
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
          stage("terraform login"){
            environment{
              TF_TOKEN_app_terraform_io= credentials('terraform_token')
            }
            steps{
              script{
                sh 'terraform login'
              }
            }
          }
          stage("terraform provisioning"){
            environment{
              TF_VAR_namespace = "monitoring"
              TF_VAR_imageName = "${IMAGE_NAME}:${APP_VERSION}"
            }
            steps{
              script{
                dir('terraform'){
                  sh 'terraform init'
                  sh 'terraform apply --auto-approve'
                }
              }
            }
          }
          stage("commit version increment"){
            environment{
              GITHUB_ACCESS_KEY = credentials('github_access_key')
            }
            steps{
              script{
                withCredentials([usernamePassword(credentialsId:'github_credentials',passwordVariable:'GIT_PASS',usernameVariable:'GIT_USER')]){
                  sh "git remote set-url origin https://${GITHUB_ACCESS_KEY}@github.com/${GIT_USER}/devops-project.git"
                  sh "git add ."
                  sh 'git commit -m "jenkins: version bump - commit"'
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
    }
}