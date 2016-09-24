// Sample spring project git url
def scmURL = "git@gitlab:${env.WORKSPACE_NAME}/spring-petclinic.git"

gitlabBuilds(builds: ["junit test & compile", "sonar code quality", "deploy to dev", "regression test", "performance test", "deploy to stage", "deploy to prod"]) {
  stage 'junit test & compile'
  node ('docker') {
    def mvnHome = tool name: 'ADOP Maven', type: 'hudson.tasks.Maven$MavenInstallation'
    gitlabCommitStatus('junit test & compile') {
      checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'adop-jenkins-master', name: 'origin', url: "${scmURL}" ]]])
      sh "${mvnHome}/bin/mvn package "
      junit '**/target/surefire-reports/TEST-*.xml'
    }
  }
  
  // define Sonar variables
  env.DB_PASSWORD = env.SONAR_DB_PASSWORD
  env.DB_USER = env.SONAR_DB_LOGIN
  
  stage 'sonar code quality'
  node ('docker') {
    def mvnHome = tool name: 'ADOP Maven', type: 'hudson.tasks.Maven$MavenInstallation' 
    env.PATH = "${mvnHome}/bin:${env.PATH}"
    gitlabCommitStatus('sonar code quality') {
        sh '''#!/bin/bash -e
        #mvn sonar:sonar -Dsonar.host.url=http://sonar:9000/sonar \
            -Dsonar.login=${INITIAL_ADMIN_USER} -Dsonar.password=${INITIAL_ADMIN_PASSWORD} \
            -Dsonar.jdbc.url='jdbc:mysql://sonar-mysql:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true' \
            -Dsonar.jdbc.username=sonar -Dsonar.jdbc.password=sonar
        '''
    }
  }

  // Define the Service application name and tags for the first Openshift environment
  env.BASE_DOCKER_IMAGE="wildfly:10.0"
  env.OC_APP_NAME="petclinic"
  env.OC_DEV_PROJECT="web-team-dev"
  env.OC_STAGE_PROJECT="web-team-stage"
  env.OC_PROD_PROJECT="web-team-prod"
  env.OC_DOWNLOAD_URL="https://github.com/openshift/origin/releases/download/v1.3.0/openshift-origin-client-tools-v1.3.0-3ab7af3d097b57f933eccef684a714f2368804e7-linux-64bit.tar.gz"
  env.OC_BINARY="openshift-origin-client-tools-v1.3.0-3ab7af3d097b57f933eccef684a714f2368804e7-linux-64bit.tar.gz"
  
  stage 'deploy to dev'
  node ('docker') {
    gitlabCommitStatus('deploy to dev') {
      withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'oc-login', passwordVariable: 'OC_PASSWORD', usernameVariable: 'OC_USER']]) {
        env.WORKSPACE=pwd()
        sh '''#!/bin/bash -e
        
        # Ensure that oc command line is installed
        if [[ ! -f /usr/bin/oc ]]; then
          echo "oc command not installed."
          cd /opt
          # Delete any openshift related installer
          rm -fr openshift* /usr/bin/oc
          
          # Install openshift commandline
          wget ${OC_DOWNLOAD_URL}
          tar -xzf ${OC_BINARY}
          cd $(ls | grep openshift)
          ln -sv $(pwd)/oc /usr/bin/oc
        fi

        cd ${WORKSPACE}
        
        # Login to Openshift
        oc login ${OC_MASTER_API_URL} -u ${OC_USER} -p ${OC_PASSWORD} --insecure-skip-tls-verify=true
        
        # Ensure that the project is created
        OC_PROJECT_DESCRIPTION="Development"
        oc new-project ${OC_DEV_PROJECT} --display-name="Web Team ${OC_PROJECT_DESCRIPTION}" \
                --description="${OC_PROJECT_DESCRIPTION} project for the web team." | true
        
        # Go to project namespace
        oc project ${OC_DEV_PROJECT}
        
        # Ensure that there is no existing deployment configuration.
        if [[ $(oc get deploymentconfigs | grep ${OC_APP_NAME} | wc -l) -eq 0 ]]; 
        then
        
          # Ensure there are no existing build configs with same app name
          oc delete all -l build=${OC_APP_NAME} | true
          
          # Create a new build 
          oc new-build -i ${BASE_DOCKER_IMAGE} --binary=true --context-dir=/ --name=${OC_APP_NAME}
          
          # Trigget the build
          oc start-build ${OC_APP_NAME} --from-dir=target/ --follow
          oc logs -f bc/${OC_APP_NAME}
          
          # Create a new application
          oc new-app -i ${OC_APP_NAME}
          
          # Expose the application http url to openshift router
          oc expose svc/${OC_APP_NAME} --path="/petclinic"
          
        else
        
          # Trigger a new build to create a new image and deploys it
          oc start-build ${OC_APP_NAME} --from-dir=target/ --follow
          
        fi
        sleep 20
        
        # Basic scripting to wait for the application url to be up and running
        APP_URL="http://${OC_APP_NAME}-${OC_DEV_PROJECT}.${OC_APPS_DOMAIN}/${OC_APP_NAME}/"
        COUNT=0
        MAX_RETRIES=10
        until [[ $( curl -I -s ${APP_URL} | head -1 | grep 200 | wc -l) -eq 1 ]] || [[ ${COUNT} -eq ${MAX_RETRIES} ]]
        do
          echo "Waiting for the petclinic ${APP_URL} to be up and running.."
          sleep 5
          let COUNT+=1
        done
        if [[ ${COUNT} -gt ${MAX_RETRIES} ]]; then echo "Waited for too long, I can't anymore.."; exit 1; fi
        
        '''
      }
    }
  }
  
  stage 'regression test'
  node ('docker') {
    def mvnHome = tool name: 'ADOP Maven', type: 'hudson.tasks.Maven$MavenInstallation'
    gitlabCommitStatus('regression test') {
      env.PATH = "${mvnHome}/bin:${env.PATH}"
      checkout changelog: false, poll: false, scm: [$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'regression-test']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'adop-jenkins-master', url: "git@gitlab:/${env.WORKSPACE_NAME}/adop-cartridge-java-regression-tests.git"]]]
      sh '''#!/bin/bash -e
      CONTAINER_NAME="owasp_zap-${gitlabSourceBranch:-main}"
      APP_URL="http://${OC_APP_NAME}-${OC_DEV_PROJECT}.${OC_APPS_DOMAIN}/petclinic"
      
      echo "Starting OWASP ZAP Intercepting Proxy"
      cd regression-test/
      docker rm -f ${CONTAINER_NAME} | true
      docker run -it -d --net=$DOCKER_NETWORK_NAME -e affinity:container==jenkins-slave \
              --name ${CONTAINER_NAME} -P nhantd/owasp_zap start zap-test
      echo "Sleeping for 30 seconds.. Waiting for OWASP Zap proxy to be up and running.."
      sleep 30
      echo "Starting Selenium test through maven.."
      mvn clean -B test -DPETCLINIC_URL=${APP_URL} -DZAP_IP=${CONTAINER_NAME} -DZAP_PORT=9090 -DZAP_ENABLED=true
      docker rm -f ${CONTAINER_NAME}
      '''
      step([$class: 'CucumberReportPublisher', fileExcludePattern: '', fileIncludePattern: '', ignoreFailedTests: false, jenkinsBasePath: '', jsonReportDirectory: 'regression-test/target', parallelTesting: false, pendingFails: false, skippedFails: false, undefinedFails: false])
    }    
  }

  stage 'openshift: scale up environment'
  node ('docker') {
      sh '''#!/bin/bash -e
      SCALE_COUNT=3
      REPLICATE_CONTROLLER_NAME=$(oc get rc -l app=${OC_APP_NAME} | tail -1 | awk '{print $1}')
      oc scale --replicas=${SCALE_COUNT} rc ${REPLICATE_CONTROLLER_NAME}
      until [[ $( oc get pods | grep ${OC_APP_NAME} | grep Running  | wc -l) -eq ${SCALE_COUNT} ]]; 
      do
         echo "Waiting for the service ${OC_APP_NAME} to be scaled up to ${SCALE_COUNT}.."
         sleep 3
      done
      sleep 5
      '''
  }
  
  stage 'performance test'
  node ('docker') {
    gitlabCommitStatus('performance test') {
      def antHome = tool name: 'ADOP Ant', type: 'hudson.tasks.Ant$AntInstallation'
      def mvnHome = tool name: 'ADOP Maven', type: 'hudson.tasks.Maven$MavenInstallation'
      env.PATH = "${antHome}/bin:${mvnHome}/bin:${env.PATH}"
      env.WORKSPACE = pwd()
      sh '''#!/bin/bash -e
      JMETER_TESTDIR=jmeter-test
      rm -fr $JMETER_TESTDIR
      mkdir -p $JMETER_TESTDIR
      cp -rp $(ls | grep -v $JMETER_TESTDIR) $JMETER_TESTDIR/
      
      if [ -e ../apache-jmeter-2.13.tgz ]; then
  	  cp ../apache-jmeter-2.13.tgz $JMETER_TESTDIR
      else
  	  wget https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-2.13.tgz
        cp apache-jmeter-2.13.tgz ../
        mv apache-jmeter-2.13.tgz $JMETER_TESTDIR
     fi
     
     cd $JMETER_TESTDIR
     PETCLINIC_HOST=${OC_APP_NAME}-${OC_PROJECT}.${OC_APPS_DOMAIN}
     tar -xf apache-jmeter-2.13.tgz
     echo 'Changing user defined parameters for jmx file'
     sed -i 's/PETCLINIC_HOST_VALUE/'"${PETCLINIC_HOST}"'/g' src/test/jmeter/petclinic_test_plan.jmx
     sed -i 's/PETCLINIC_PORT_VALUE/80/g' src/test/jmeter/petclinic_test_plan.jmx
     sed -i 's/CONTEXT_WEB_VALUE/petclinic/g' src/test/jmeter/petclinic_test_plan.jmx
     sed -i 's/HTTPSampler.path"></HTTPSampler.path">petclinic</g' src/test/jmeter/petclinic_test_plan.jmx
     
     cd ../
     ant -f $JMETER_TESTDIR/apache-jmeter-2.13/extras/build.xml \
            -Dtestpath=${WORKSPACE}/${JMETER_TESTDIR}/src/test/jmeter -Dtest=petclinic_test_plan
     
     cd ${WORKSPACE}/src/test/gatling
     sed -i "s+###TOKEN_VALID_URL###+http://${PETCLINIC_HOST}+g" src/test/scala/default/RecordedSimulation.scala
     sed -i "s/###TOKEN_RESPONSE_TIME###/10000/g" src/test/scala/default/RecordedSimulation.scala
     mvn gatling:execute
     '''
     
     step([$class: 'GatlingPublisher', enabled: true])
     publishHTML(target: [allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'jmeter-test/src/test/jmeter', reportFiles: 'petclinic_test_plan.html', reportName: 'Jmeter Report'])
     
     // Scale Down the service
     sh '''#!/bin/bash -e
     APP_NAME=dev-petclinic
     REPLICATE_CONTROLLER_NAME=$(oc get rc -l app=${OC_APP_NAME} | tail -1 | awk '{print $1}')
     oc scale --replicas=1 rc ${REPLICATE_CONTROLLER_NAME}
     '''
    }
  }
  
  stage 'deploy to stage'
  node ('docker') {
    gitlabCommitStatus('deploy to stage') {
      withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'oc-login', passwordVariable: 'OC_PASSWORD', usernameVariable: 'OC_USER']]) {
        sh '''#!/bin/bash -e
    
        # Login to Openshift
        oc login ${OC_MASTER_API_URL} -u ${OC_USER} -p ${OC_PASSWORD} --insecure-skip-tls-verify=true
        
        # Ensure that project is created
        OC_PROJECT_DESCRIPTION="Stage Environment"
        oc new-project ${OC_STAGE_PROJECT} --display-name="Web Team ${OC_PROJECT_DESCRIPTION}" \
              --description="${OC_PROJECT_DESCRIPTION} project for the web team." | true
              
        # Go to project namespace
        oc project ${OC_STAGE_PROJECT}
        
        # Ensure that stage project can pull images from dev project
        oc policy add-role-to-user system:image-puller system:serviceaccount:${OC_STAGE_PROJECT}:default --namespace=${OC_DEV_PROJECT}
        
        # Ensure that there is no existing deployment configuration
        if [[ $(oc get deploymentconfigs | grep ${OC_APP_NAME} | wc -l) -eq 0 ]]; 
        then
          
          # Deploy from the latest docker image from Dev
          oc new-app -i ${OC_DEV_PROJECT}/${OC_APP_NAME}:latest --name=${OC_APP_NAME}
          
          # Expose the application http url to openshift router
          oc expose svc/${OC_APP_NAME} --path="/petclinic"
          
        else
        
          # Start a new deployment
          oc deploy ${OC_APP_NAME} --latest
          
        fi
        sleep 20
        
        # Basic scripting to wait for the application url to be up and running
        APP_URL="http://${OC_APP_NAME}-${OC_STAGE_PROJECT}.${OC_APPS_DOMAIN}/${OC_APP_NAME}/"
        COUNT=0
        MAX_RETRIES=10
        until [[ $( curl -I -s ${APP_URL} | head -1 | grep 200 | wc -l) -eq 1 ]] || [[ ${COUNT} -eq ${MAX_RETRIES} ]]
        do
          echo "Waiting for the petclinic ${APP_URL} to be up and running.."
          sleep 5
          let COUNT+=1
        done
        if [[ ${COUNT} -gt ${MAX_RETRIES} ]]; then echo "Waited for too long, I can't anymore.."; exit 1; fi
        
        '''
      }
    }
  }
  
  stage 'approval: deploy to prod?'
  timeout(time:1, unit:'DAYS') {
    input message:'Approve Deployment?', submitter: 'administrators'
  }
  
  stage 'deploy to prod'
  node ('docker') {
    gitlabCommitStatus('deploy to prod') {
      withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'oc-login', passwordVariable: 'OC_PASSWORD', usernameVariable: 'OC_USER']]) {
        sh '''#!/bin/bash -e
    
        # Login to Openshift
        oc login ${OC_MASTER_API_URL} -u ${OC_USER} -p ${OC_PASSWORD} --insecure-skip-tls-verify=true
        
        # Ensure that project is created
        OC_PROJECT_DESCRIPTION="Production Environment"
        oc new-project ${OC_PROD_PROJECT} --display-name="Web Team ${OC_PROJECT_DESCRIPTION}" \
              --description="${OC_PROJECT_DESCRIPTION} project for the web team." | true
        
        # Go to project namespace
        oc project ${OC_PROD_PROJECT}
        
        # Ensure that prod project can pull images from dev project
        oc policy add-role-to-user system:image-puller system:serviceaccount:${OC_PROD_PROJECT}:default --namespace=${OC_DEV_PROJECT}
        
        # Ensure that there is no existing deployment configuration
        if [[ $(oc get deploymentconfigs | grep ${OC_APP_NAME} | wc -l) -eq 0 ]]; 
        then
        
          # Deploy from the latest docker image from Dev
          oc new-app -i ${OC_DEV_PROJECT}/${OC_APP_NAME}:latest --name=${OC_APP_NAME}
          
          # Expose the application http url to openshift router
          oc expose svc/${OC_APP_NAME} --path="/petclinic"
          
        else
        
          # Start a new deployment
          oc deploy ${OC_APP_NAME} --latest
          
        fi
        sleep 20
        
        # Basic scripting to wait for the application url to be up and running
        APP_URL="http://${OC_APP_NAME}-${OC_PROD_PROJECT}.${OC_APPS_DOMAIN}/${OC_APP_NAME}/"
        COUNT=0
        MAX_RETRIES=10
        until [[ $( curl -I -s ${APP_URL} | head -1 | grep 200 | wc -l) -eq 1 ]] || [[ ${COUNT} -eq ${MAX_RETRIES} ]]
        do
          echo "Waiting for the petclinic ${APP_URL} to be up and running.."
          sleep 5
          let COUNT+=1
        done
        if [[ ${COUNT} -gt ${MAX_RETRIES} ]]; then echo "Waited for too long, I can't anymore.."; exit 1; fi
        
        '''
      }
    }
  }
  
}
