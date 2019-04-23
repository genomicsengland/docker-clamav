#!/usr/bin/env groovy

String apiVersion = "v1.0"

pipeline {
  agent {
    label 'mavenpython'
  }
  environment {
    ARTIFACTORY =  'cs-prod-tools-artifactory-01.ngis.zone:5011'
  }

  stages {
    stage('Tell Openshift to build the container and push to artifactory'){
      steps {
        script {
          try {
            echo 'This would be so much easier with a docker build agent'
            sh """
              oc project optum-patientchoice-build
              oc new-build --binary=true --name=clam-av-build --to-docker=true --to=$ARTIFACTORY/choice-service-clamav:$apiVersion --push-secret=art-nonprod-servac1
              oc start-build clam-av-build --from-dir=. --follow=true --wait
            """
          } catch (err) {
            echo "Error caught in step. Deleting BC on OpenShift."
            echo "Caught: ${err}"
            currentBuild.result = 'FAILURE'
          } finally {
            //delete the created build config to keep OSE clean for demo purposes
            //Note: If the build config is kept, then "oc new-build" is not needed for subsequent builds.
            sh "oc delete bc/clam-av-build"
          }
        }
      }
    }
    stage('Promote to E2E if Integration tests successful'){
      steps {
        script {
          echo 'Attempting curl command to trigger a copy into E2E namespace...'
          sh """
            set -x
            STATUS=\$(curl --silent --output /dev/null -w '%{http_code}' \\
              -x http://10.252.0.130:3128 -i -uose-ngis-art-nonprod:Password1 \\
              -X POST "https://cs-prod-tools-artifactory-01.gel.zone/artifactory/api/docker/ngis-build/v2/promote" \\
              -H "Content-Type: application/json" -d '{"targetRepo":"ngis-e2e","dockerRepository":"choice-service-clamav/${apiVersion}", "copy":"true"}')
            if [ \$STATUS -eq 200 ]; then
                echo "Promotion ended successfully"
                exit 0
            fi
                echo "Promotion failed"
                exit 1
            done
          """
        }
      }
    }
  }

  post('Publish Report & Metrics') {
    always {
      archive 'Jenkinsfile'
    }
  }
}
