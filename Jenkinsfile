//
// Copyright © 2018 Province of British Columbia
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
// http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
//

//JENKINS DEPLOY ENVIRONMENT VARIABLES:
// - JENKINS_JAVA_OVERRIDES  -Dhudson.model.DirectoryBrowserSupport.CSP= -Duser.timezone=America/Vancouver
//   -> user.timezone : set the local timezone so logfiles report correxct time
//   -> hudson.model.DirectoryBrowserSupport.CSP : removes restrictions on CSS file load, thus html pages of test reports are displayed pretty
//   See: https://docs.openshift.com/container-platform/3.9/using_images/other_images/jenkins.html for a complete list of JENKINS env vars
// - SLACK_TOKEN

// define constants
def BUILDCFG_NAME ='myapp'
def IMAGE_NAME = 'myapp'
def DEV_DEPLOYMENT_NAME = 'myapp-dev'
def DEV_TAG_NAME = 'dev'
def DEV_NS = 'ocp-myapp-dev'
def TST_DEPLOYMENT_NAME = 'myapp-test'
def TST_TAG_NAME = 'test'
def TST_BCK_TAG_NAME = 'test-previous'
def TST_NS = 'ocp-mayapp-test'
def PROD_DEPLOYMENT_NAME = 'mayapp'
def PROD_TAG_NAME = 'prod'
def PROD_BCK_TAG_NAME = 'prod-previous'
def PROD_NS = 'ocp-myapp-prod'

// define groovy functions

// send a msg to slack channel that deploy occured
import groovy.json.JsonOutput
def notifySlack(text, channel, url, attachments) {
    def slackURL = url
    def jenkinsIcon = 'https://wiki.jenkins-ci.org/download/attachments/2916393/logo.png'
    def payload = JsonOutput.toJson([text: text,
        channel: channel,
        username: "Jenkins",
        icon_url: jenkinsIcon,
        attachments: attachments
    ])
    def encodedReq = URLEncoder.encode(payload, "UTF-8")
    sh("curl -s -S -X POST " +
            "--data \'payload=${encodedReq}\' ${slackURL}")    
}

// create a string listing commit msgs occured since last build
@NonCPS
def getChangeString() {
  MAX_MSG_LEN = 512
  def changeString = ""
  def changeLogSets = currentBuild.changeSets
  for (int i = 0; i < changeLogSets.size(); i++) {
     def entries = changeLogSets[i].items
     for (int j = 0; j < entries.length; j++) {
         def entry = entries[j]
         truncated_msg = entry.msg.take(MAX_MSG_LEN)
         changeString += " - ${truncated_msg} [${entry.author}]\n"
     }
  }
  if (!changeString) {
     changeString = "No changes"
  }
  return changeString
}

// pipeline

// Note: openshiftVerifyDeploy requires policy to be added:
// oc policy add-role-to-user view system:serviceaccount:devex-platform-tools:jenkins -n devex-platform-dev
// oc policy add-role-to-user view system:serviceaccount:devex-platform-tools:jenkins -n devex-platform-test
// oc policy add-role-to-user view system:serviceaccount:devex-platform-tools:jenkins -n devex-platform-prod

// define job properties - keep 10 builds only
properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10']]])

// Part 1 - CI - Source code scanning, build, dev deploy

node('maven') {

    stage('checkout') {
       echo "checking out source"
       echo "Build: ${BUILD_ID}"
       checkout scm
    }
    stage('dependency check') {
          dir('owasp') {
            // content of 
            // http://dl.bintray.com/jeremy-long/owasp/dependency-check-3.1.2-release.zip'
            // expected in git repo in owasp/
            sh './dependency-check/bin/dependency-check.sh --project "MyApp" --scan ../package.json --enableExperimental --enableRetired'
            sh 'rm -rf ./dependency-check/data/'
            publishHTML (target: [
                                allowMissing: false,
                                alwaysLinkToLastBuild: false,
                                keepAll: true,
                                reportDir: './',
                                reportFiles: 'dependency-check-report.html',
                                reportName: "OWASP Dependency Check Report"
                          ])
          }
    }
    stage('code quality check') {
           SONARQUBE_URL = sh (
               script: 'oc get routes -o wide --no-headers | awk \'/sonarqube/{ print match($0,/edge/) ?  "https://"$2 : "http://"$2 }\'',
               returnStdout: true
                  ).trim()
           echo "SONARQUBE_URL: ${SONARQUBE_URL}"
           dir('sonar-runner') {
            sh returnStdout: true, script: "./gradlew sonarqube -Dsonar.host.url=${SONARQUBE_URL} -Dsonar.verbose=true --stacktrace --info -Dsonar.projectName=Devex.Dev -Dsonar.branch=develop -Dsonar.projectKey=org.sonarqube:bcgov-devex-dev -Dsonar.sources=.."
           }
    }

    stage('build') {
	    echo "Building..."
	    openshiftBuild bldCfg: BUILDCFG_NAME, verbose: 'false', showBuildLogs: 'true'
            sleep 5
	    // openshiftVerifyBuild bldCfg: BUILDCFG_NAME
            echo ">>> Get Image Hash"
            IMAGE_HASH = sh (
              script: """oc get istag ${IMAGE_NAME}:latest -o template --template=\"{{.image.dockerImageReference}}\"|awk -F \":\" \'{print \$3}\'""",
                returnStdout: true).trim()
            echo ">> IMAGE_HASH: ${IMAGE_HASH}"
	    echo ">>>> Build Complete"
    }
    stage('Dev deploy') {
	    echo ">>> Tag ${IMAGE_HASH} with ${DEV_TAG_NAME}"
 	    openshiftTag destStream: IMAGE_NAME, verbose: 'false', destTag: DEV_TAG_NAME, srcStream: IMAGE_NAME, srcTag: "${IMAGE_HASH}"
            sleep 5
	    openshiftVerifyDeployment depCfg: DEV_DEPLOYMENT_NAME, namespace: DEV_NS, replicaCount: 1, verbose: 'false', verifyReplicaCount: 'false'
	    echo ">>>> Deployment Complete"
	    // send msg to slack
	    def attachment = [:]
            attachment.fallback = 'See build log for more details'
            attachment.title = "Build ${BUILD_ID} OK! :heart: :tada:"
            attachment.color = '#00FF00' // Lime Green
            attachment.text = "Changes applied:\n" + getChangeString() + "\nCommit ${GIT_COMMIT_SHORT_HASH} by ${GIT_COMMIT_AUTHOR}"
	    notifySlack("Dev Deploy", "#builds", "https://hooks.slack.com/services/${SLACK_TOKEN}", attachment)
    }
}

// Part 2 - Security scan of deployed app in dev

// ensure pod labels/names are unique
def zappodlabel = "myapp-zap-${UUID.randomUUID().toString()}"
podTemplate(label: zappodlabel, name: zappodlabel, serviceAccount: 'jenkins', cloud: 'openshift', containers: [
  containerTemplate(
    name: 'jnlp',
    image: '172.50.0.2:5000/openshift/jenkins-slave-zap',
    resourceRequestCpu: '500m',
    resourceLimitCpu: '1000m',
    resourceRequestMemory: '3Gi',
    resourceLimitMemory: '4Gi',
    workingDir: '/home/jenkins',
    command: '',
    args: '${computer.jnlpmac} ${computer.name}'
  )
]) {
     stage('ZAP Security Scan') {
        node(zappodlabel) {
          sleep 60
          def retVal = sh returnStatus: true, script: '/zap/zap-baseline.py -r baseline.html -t http://myapp-dev.pathfinder.gov.bc.ca/'
          publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true, reportDir: '/zap/wrk', reportFiles: 'baseline.html', reportName: 'ZAP Baseline Scan', reportTitles: 'ZAP Baseline Scan'])
          echo "Return value is: ${retVal}"
        }
     }
  }

// Part 3 - BDD Testing 

stage('Functional Test Dev') {
  // ensure pod labels/names are unique
  def bddpodlabel = "myapp-bdd-${UUID.randomUUID().toString()}"	
  podTemplate(label: bddpodlabel, name: bddpodlabel, serviceAccount: 'jenkins', cloud: 'openshift', 
    volumes: [
	    emptyDirVolume(mountPath:'/dev/shm', memory: true)
    ],
    containers: [
      containerTemplate(
        name: 'jnlp',
        image: '172.50.0.2:5000/openshift/jenkins-slave-bddstack',
        resourceRequestCpu: '500m',
        resourceLimitCpu: '1000m',
        resourceRequestMemory: '1Gi',
        resourceLimitMemory: '4Gi',
        workingDir: '/home/jenkins',
        command: '',
        args: '${computer.jnlpmac} ${computer.name}',
        envVars: [
          envVar(key:'BASEURL', value: 'http://myapp-dev.pathfinder.gov.bc.ca/')
        ]
      )]
  ) {
    node(bddpodlabel) {
      //the checkout is mandatory, otherwise functional test would fail
      echo "checking out source"
      checkout scm
      dir('functional-tests') {
        try {
          sh './gradlew chromeHeadlessTest'
        } finally { 
          archiveArtifacts allowEmptyArchive: true, artifacts: 'build/reports/**/*'
          archiveArtifacts allowEmptyArchive: true, artifacts: 'build/test-results/**/*'
          junit 'build/test-results/**/*.xml'
          publishHTML (target: [
                                allowMissing: false,
                                alwaysLinkToLastBuild: false,
                                keepAll: true,
                                reportDir: 'build/reports/spock',
                                reportFiles: 'index.html',
                                reportName: "BDD Spock Report"
                          ])
          publishHTML (target: [
                                allowMissing: false,
                                alwaysLinkToLastBuild: false,
                                keepAll: true,
                                reportDir: 'build/reports/tests/chromeHeadlessTest',
                                reportFiles: 'index.html',
                                reportName: "Full Test Report"
                          ])  
	  }
        }
     }
   }
}
	
stage('deploy-test') {	
  timeout(time: 1, unit: 'DAYS') {
	  input message: "Deploy to test?", submitter: 'admin,user-view,user1-admin'
  }
  node('master') {
	  echo ">>> Tag ${TST_TAG_NAME} with ${TST_BCK_TAG_NAME}"
	  openshiftTag destStream: IMAGE_NAME, verbose: 'false', destTag: TST_BCK_TAG_NAME, srcStream: IMAGE_NAME, srcTag: TST_TAG_NAME
          echo ">>> Tag ${IMAGE_HASH} with ${TST_TAG_NAME}"
	  openshiftTag destStream: IMAGE_NAME, verbose: 'false', destTag: TST_TAG_NAME, srcStream: IMAGE_NAME, srcTag: "${IMAGE_HASH}"
          sleep 5
	  openshiftVerifyDeployment depCfg: TST_DEPLOYMENT_NAME, namespace: TST_NS, replicaCount: 1, verbose: 'false', verifyReplicaCount: 'false'
	  echo ">>>> Deployment Complete"
	  notifySlack("Test Deploy, changes:\n" + getChangeString(), "#builds", "https://hooks.slack.com/services/${SLACK_TOKEN}", [])
  }
}

stage('deploy-prod') {
    timeout(time: 3, unit: 'DAYS') {
	  input message: "Deploy to test?", submitter: 'admin,user-view,user1-admin'
    }
    node('master') {
      echo ">>> Tag ${PROD_TAG_NAME} with ${PROD_BCK_TAG_NAME}"
      openshiftTag destStream: IMAGE_NAME, verbose: 'false', destTag: PROD_BCK_TAG_NAME, srcStream: IMAGE_NAME, srcTag: PROD_TAG_NAME
      echo ">>> Tag ${IMAGE_HASH} with ${PROD_TAG_NAME}"
      openshiftTag destStream: IMAGE_NAME, verbose: 'false', destTag: PROD_TAG_NAME, srcStream: IMAGE_NAME, srcTag: "${IMAGE_HASH}"
      sleep 5
      openshiftVerifyDeployment depCfg: PROD_DEPLOYMENT_NAME, namespace: PROD_NS, replicaCount: 1, verbose: 'false', verifyReplicaCount: 'false'
      echo ">>>> Deployment Complete"
      notifySlack("PRODUCTION Deploy, changes:\n" + getChangeString(), "#builds", "https://hooks.slack.com/services/${SLACK_TOKEN}", [])
    }
}

