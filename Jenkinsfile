#!/usr/bin/groovy
@Library("ekzero-custom-libraries")

def project_name = 'vegitone'
def project_service_name = 'store-api'
def buildLabel = "slave.${env.JOB_NAME}.${env.BUILD_NUMBER}".replace('-', '_').replace('/', '_')
def awsRegion = "ca-central-1"
def imageRepo = "125968943346.dkr.ecr.ca-central-1.amazonaws.com"
def chart_name = "vegitone/vegitone-app"
// IF BRANCH is PR then do not split it
def ENVIRONMENT_NAME = (env.BRANCH_NAME.startsWith("PR")) ? env.BRANCH_NAME : extractEnvName(env.BRANCH_NAME)

def java_opts = ""

podTemplate(label: buildLabel, serviceAccount: "helm-deployer", nodeSelector: "env: jenkins-build",containers: [
    containerTemplate(name: 'maven-builder', image: 'maven:3.6.3-amazoncorretto-11', command: 'cat', ttyEnabled: true,),
    containerTemplate(name: 'build-container', image: 'docker:dind', command: 'dockerd', ttyEnabled: true, privileged: true,),
    containerTemplate(name: 'deployer', alwaysPullImage: true, image: imageRepo + '/vegitone-devops:helm-deployer', command: 'cat', ttyEnabled: true,)
],
) {
    node(buildLabel) {
        deleteDir()

        try{
            stage('Checkout') {
                echo "Checkout Started"
                myRepo = checkout scm
                gitRepoUrl = myRepo.GIT_URL.replace('https://', '')
                gitCommit = myRepo.GIT_COMMIT
                gitBranch = myRepo.GIT_BRANCH
                shortGitCommit = "${gitCommit[0..10]}"
                imageTag = ENVIRONMENT_NAME + '-' + shortGitCommit + '-' + env.BUILD_ID
                authorName = sh(script: "git log -1 --pretty=%an ${gitCommit}", returnStdout: true).trim()
                sh "rm src/main/resources/application* || ls -la"
                withAWS(region:'ca-central-1',credentials:"${aws_agent}") {
                    s3Download(file: 'src/main/resources/application.properties', bucket: 'vegitone-env', path: "${project_name}-${project_service_name}/${ENVIRONMENT_NAME}/application.properties")
                }
                echo "Checkout Done"
            }

            stage ("Code Analysis") {
                if (ENVIRONMENT_NAME.startsWith("PR")){
                    withSonarQubeEnv('My SonarQube Server') {
                        sh "mvn clean package org.sonarsource.scanner.maven:sonar-maven-plugin:3.7.0.1746:sonar"
                        // timeout(time: 3, unit: 'MINUTES'){
                        //     def qg = waitForQualityGate()
                        //     if (qg.status != 'OK') {
                        //         error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        //     }
                        // }
                    }
                }
            }

            stage ('Building Application') {
                container('maven-builder') {
                    echo "Java Application Build Started"
                    sh 'mvn clean install -DskipTests -ntp'
                    echo "Java Application Built Successfully"
                }
            }

            stage('Docker Image Build and Publish') {
                container('build-container') {
                    if (ENVIRONMENT_NAME == "dev") {
                        java_opts = "-Xms400M -Xmx500M"
                    } else if (ENVIRONMENT_NAME == "qa") {
                        java_opts = "-Xms4000M -Xmx500M"
                    } else if (ENVIRONMENT_NAME == "prod") {
                        java_opts = "-Xms600M -Xmx900M"
                    } else {
                        java_opts = ""
                    }
                    echo "Docker Image Build Started"
                    def app = docker.build("${imageRepo}/${project_name}-${project_service_name}:${imageTag}", "--build-arg java_opts=\"${java_opts}\" .")
                    echo "Docker Image Push Started"
                    docker.withRegistry("https://${imageRepo}","ecr:ca-central-1:${aws_agent}") {
                        app.push()
                    }
                    echo "Docker Image Pushed"
                }
            }

            stage('Publish') {
                container('deployer') {
                    sh "helm repo add vegitone s3://vegitone-helm-repo/charts"
                    sh "helm upgrade --install --create-namespace --namespace ${project_name}-${ENVIRONMENT_NAME} ${project_name}-${project_service_name} ${chart_name} --set image.tag=${imageTag}  -f deploy/${ENVIRONMENT_NAME}.yaml"
                    postAlways()
                }
            }
        }
        catch (Exception err) {       
            postFailure(err)            
            error("Pipeline failed. ERROR: " + err)
        }
    }
}

def postFailure(e) {
    msg =  "<b><font color='#ff0000'>failed due to : ${e}</font></b>"
    sendNotification("<b><font color='#ff0000'>FAILED</font></b>", msg)
}

def postAlways() {
    msg =  'Successfully deployed'
    sendNotification("<b><font color='#006400'>SUCCESS</font></b>", msg)
}

def sendNotification(String status, String msg) {

    String google_chat_url = "${vegitone_gchat_url}"
    String branch_name = env.JOB_NAME.split('/')[1]
    String build_number = "${env.BUILD_NUMBER}"
    String job_name = env.JOB_NAME.split('/')[0]
    String job_url = "${env.BUILD_URL}console"
    googleChat.sendNotification(google_chat_url,job_name,branch_name,build_number,status,gitCommit,authorName,job_url,gitRepoUrl,msg)
}


def extractEnvName(branchName) {
    return branchName.split('/')[1]
}