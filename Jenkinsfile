@Library('dynatrace@master') _

pipeline {
  parameters {
    string(name: 'APP_NAME', defaultValue: '', description: 'The name of the service to deploy.', trim: true)
    string(name: 'TAG_STAGING', defaultValue: '', description: 'The image of the service to deploy.', trim: true)
    string(name: 'VERSION', defaultValue: '', description: 'The version of the service to deploy.', trim: true)
  }
   environment {
      APP_NAME = "${env.APP_NAME}"
      ARTEFACT_ID = "sockshop-" + "${env.APP_NAME}"
      DYNATRACEID="${env.DT_ACCOUNTID}"
      DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
      NLAPIKEY="${env.NL_WEB_API_KEY}"
      NL_DT_TAG="app:${env.APP_NAME},environment:${env.TAG_STAGING}"
      OUTPUTSANITYCHECK="$WORKSPACE/infrastructure/sanitycheck.json"
      NEOLOAD_ASCODEFILE="$WORKSPACE/neoload/e2e_neoload.yaml"
      NEOLOAD_ANOMALIEDETECTIONFILE="$WORKSPACE/monspec/e2e_anomalieDection.json"
      GITORIGIN="neotyslab"
    }
  agent {
    label 'kubegit'
  }
  stages {
    stage('Update Deployment and Service specification') {
      steps {
        container('git') {
          withCredentials([usernamePassword(credentialsId: 'git-credentials-acm', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
            sh "git config --global user.email ${env.GITHUB_USER_EMAIL}"
            sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/k8s-deploy-staging"
            sh "cd k8s-deploy-staging/ && sed -i 's#image: .*#image: ${env.TAG_STAGING}#' ${env.APP_NAME}.yml"
            sh "cd k8s-deploy-staging/ && git add ${env.APP_NAME}.yml && git commit -m 'Update ${env.APP_NAME} version ${env.VERSION}'"
            sh "cd k8s-deploy-staging/ && git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/k8s-deploy-staging"
          }
        }
      }
    }
    stage('Deploy to staging namespace') {
      steps {
        checkout scm
        container('kubectl') {
          sh "kubectl -n staging apply -f ${env.APP_NAME}.yml"
        }
      }
    }
    /*
    stage('DT Deploy Event') {
      steps {
        container("curl") {
          // send custom deployment event to Dynatrace
          sh "curl -X POST \"$DT_TENANT_URL/api/v1/events?Api-Token=$DT_API_TOKEN\" -H \"accept: application/json\" -H \"Content-Type: application/json\" -d \"{ \\\"eventType\\\": \\\"CUSTOM_DEPLOYMENT\\\", \\\"attachRules\\\": { \\\"tagRule\\\" : [{ \\\"meTypes\\\" : [\\\"SERVICE\\\"], \\\"tags\\\" : [ { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"app\\\", \\\"value\\\" : \\\"${env.APP_NAME}\\\" }, { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"environment\\\", \\\"value\\\" : \\\"staging\\\" } ] }] }, \\\"deploymentName\\\":\\\"${env.JOB_NAME}\\\", \\\"deploymentVersion\\\":\\\"${env.VERSION}\\\", \\\"deploymentProject\\\":\\\"\\\", \\\"ciBackLink\\\":\\\"${env.BUILD_URL}\\\", \\\"source\\\":\\\"Jenkins\\\", \\\"customProperties\\\": { \\\"Jenkins Build Number\\\": \\\"${env.BUILD_ID}\\\",  \\\"Git commit\\\": \\\"${env.GIT_COMMIT}\\\" } }\" "
        }
      }
    }
    */
    stage('Start NeoLoad infrastructure') {

            steps {
                    container('kubectl') {
                        script {
                         sh "kubectl create -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml"
                        }
                    }
            }

    }
    stage('Run production ready e2e check in staging') {
      steps {
        echo "Waiting for the service to start..."
        sleep 250

        recordDynatraceSession(
          envId: 'Dynatrace Tenant',
          testCase: 'loadtest',
          tagMatchRules: [
            [
              meTypes: [
                [meType: 'SERVICE']
              ],
              tags: [
                [context: 'CONTEXTLESS', key: 'app', value: "${env.APP_NAME}"],
                [context: 'CONTEXTLESS', key: 'environment', value: 'staging']
              ]
            ]
          ]
        ) 
        {
          sh "sed -i 's/HOST_TO_REPLACE/front-end.staging.svc/'  ${NEOLOAD_ASCODEFILE}"
          sh "sed -i 's/PORT_TO_REPLACE/8080/'  ${NEOLOAD_ASCODEFILE}"
          sh "sed -i 's/DTID_TO_REPLACE/${DYNATRACEID}/'  ${NEOLOAD_ASCODEFILE}"
          sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  ${NEOLOAD_ASCODEFILE}"
          sh "sed -i 's,JSONFILE_TO_REPLACE,${NEOLOAD_ANOMALIEDETECTIONFILE},'  ${NEOLOAD_ASCODEFILE}"
          sh "sed -i 's/TAGS_TO_REPLACE/${NL_DT_TAG}/'  ${NEOLOAD_ASCODEFILE}"
          sh "sed -i 's,OUTPUTFILE_TO_REPLACE,${OUTPUTSANITYCHECK},'  ${NEOLOAD_ASCODEFILE}"

          container('neoload') {
            script {
              sh "mkdir -p /home/jenkins/.neotys/neoload"
              sh "cp $WORKSPACE/infrastructure/infrastructure/neoload/license.lic /home/jenkins/.neotys/neoload/"

              status =sh(script:"/neoload/bin/NeoLoadCmd -project $WORKSPACE/neoload/load_template/load_template.nlp ${NEOLOAD_ASCODEFILE} -testResultName e2eCheck__${BUILD_NUMBER} -description e2eCheck__${BUILD_NUMBER} -nlweb -L End2End=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -launch End2End -noGUI", returnStatus: true)


              if (status != 0) {
                currentBuild.result = 'FAILED'
                error "Production ready e2e check in staging failed."
              }
            }
          }
        }

        perfSigDynatraceReports(
          envId: 'Dynatrace Tenant', 
          nonFunctionalFailure: 1, 
          specFile: "monspec/e2e_perfsig.json"
        )
      }
    }
  }
  post {
            always {
              container('kubectl') {
                     script {
                      echo "delete neoload infrastructure"
                      sh "kubectl delete svc nl-lg-e2e -n cicd-neotys"
                      sh "kubectl delete pod nl-lg-e2e -n cicd-neotys --grace-period=0 --force"
                     }
              }
            }

          }
}
