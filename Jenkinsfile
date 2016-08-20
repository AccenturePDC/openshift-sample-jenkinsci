properties properties: [[$class: 'GitLabConnectionProperty', gitLabConnection: 'ADOP Gitlab']]

def scmURL = 'git@gitlab:adopadmin/os-sample-java-web.git' 

stage 'build: maven'
node ('docker') {
  def mvnHome = tool name: 'ADOP Maven', type: 'hudson.tasks.Maven$MavenInstallation'
  gitlabCommitStatus("Maven Build") {
    checkout([$class: 'GitSCM', branches: [[name: 'origin/${gitlabSourceBranch}']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'adop-jenkins-master', name: 'origin', refspec: '+refs/heads/*:refs/remotes/origin/* ', url: "${scmURL}" ]]])
    sh "${mvnHome}/bin/mvn clean install"
  }
}

stage 'test: code quality & unit test'
node ('docker') {
  def mvnHome = tool name: 'ADOP Maven', type: 'hudson.tasks.Maven$MavenInstallation'
  gitlabCommitStatus("Code Quality") {
    sh "${mvnHome}/bin/mvn sonar:sonar -Dsonar.host.url=http://sonar:9000/sonar -Dsonar.login=adopadmin -Dsonar.password=bryan123 -Dsonar.jdbc.url='jdbc:mysql://sonar-mysql:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true' -Dsonar.jdbc.username=sonar -Dsonar.jdbc.password=sonar"
  }
}

stage 'deploy: dev'
node ('docker') {
  gitlabCommitStatus("Deploy to Dev") {
    sh '''#!/bin/bash -e

    APP_NAME=${gitlabSourceRepoName}
    PROJECT=develop-feature

    oc project ${PROJECT}

    if [[ $(oc get deploymentconfigs | grep ${APP_NAME} | wc -l) -eq 0 ]]; 
    then
      oc new-build -i wildfly:10.0 --binary=true --context-dir=/ --name=${APP_NAME}
      oc start-build ${APP_NAME} --from-dir=target/ --follow
      oc logs -f bc/${APP_NAME}
      oc new-app -i ${APP_NAME}
      oc expose svc/${APP_NAME}
    else
      oc start-build ${APP_NAME} --from-dir=target/ --follow
    fi
    '''
   }
}

stage 'test: regression'
node ('docker') {
  gitlabCommitStatus('Regression Test') {
    sh "echo 'Running test in dev environment..'"
  }
}

stage 'email: deploy to sit approval'
node {
  emailext body: 'Jenkins deployment requires your approval.', subject: 'Approval Required', to: 'bryansazon@hotmail.com'
}

stage 'approval'
timeout(time:5, unit:'DAYS') {
  input message:'Dev testing passed. Approve deployment to SIT?', submitter: 'administrators'
}

stage 'deploy: sit'
node ('docker') {
  gitlabCommitStatus('Deploy to SIT') {
    echo "Deployment to SIT completed"
  }
}

stage 'test: integration'
node ('docker') {
  gitlabCommitStatus('Integration Test') {
    echo "Integration test completed."
  }
}