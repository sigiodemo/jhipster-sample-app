pipeline {
  agent any

  environment {
    PROJECT = 'jhipster'
    VERSION = '1.0'
    BRANCH = 'main'
    POLARIS_ACCESS_TOKEN = credentials('_POLARIS_TOKEN')
    BLACKDUCK_ACCESS_TOKEN = credentials('_BLACKDUCK_TOKEN')
    IO_TOKEN = credentials('_IO_TOKEN')
    CODEDX_TOKEN = credentials('_CODEDX_TOKEN')
    GITHUB_TOKEN = credentials('_GITHUB_TOKEN')
    GITHUB_USER = "sigiodemo"
    GITHUB_PROJECT = "jhipster-sample-app"
    SEEKER_TOKEN = credentials('_SEEKER_TOKEN')
    SERVER_START = "java -javaagent:seeker/seeker-agent.jar -jar target/jhipster-sample-application-0.0.1-SNAPSHOT.jar"
    SERVER_STRING = "Application 'jhipsterSampleApplication' is running!"
    SERVER_WORKINGDIR = ""
    SEEKER_RUN_TIME = 180
    SEEKER_PROJECT_KEY = 'jhip'
  }

  stages{
    stage('NPM Install') {
      steps {
        sh 'npm install'
      }
    }

    stage('Build') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }
    stage('Set Up Environment') {
      steps {
        sh '''
          curl -s -L https://raw.githubusercontent.com/jones6951/io-scripts/main/getProjectID.sh > /tmp/getProjectID.sh
          curl -s -L https://raw.githubusercontent.com/jones6951/io-scripts/main/serverStart.sh > /tmp/serverStart.sh
          curl -s -L https://raw.githubusercontent.com/jones6951/io-scripts/main/isNumeric.sh > /tmp/isNumeric.sh
          curl -s -L https://raw.githubusercontent.com/synopsys-sig/io-artifacts/main/prescription.sh > /tmp/prescription.sh

          chmod +x /tmp/getProjectID.sh
          chmod +x /tmp/serverStart.sh
          chmod +x /tmp/isNumeric.sh
          chmod +x /tmp/prescription.sh
        '''
      }
    }

    stage('IO - Get Prescription') {
        steps {
        echo "Getting Prescription"
        sh '''
          projectID=$(/tmp/getProjectID.sh --url=${_CODEDX_SERVER_URL} --apikey=${CODEDX_TOKEN} --project=${PROJECT})
          /tmp/prescription.sh \
            --stage="IO" \
            --persona="devsecops" \
            --io.url="${_IO_SERVER_URL}/api/ioiq/" \
            --io.token="${IO_TOKEN}" \
            --manifest.type="json" \
            --project.name="${PROJECT}" \
            --asset.id="${PROJECT}" \
            --workflow.url="${_IO_SERVER_URL}/api/workflowengine/" \
            --workflow.version="2022.4.1" \
            --file.change.threshold="10" \
            --sast.rescan.threshold="20" \
            --sca.rescan.threshold="20" \
            --scm.type="github" \
            --scm.owner="${GITHUB_USER}" \
            --scm.repo.name="${GITHUB_PROJECT}" \
            --scm.branch.name="${BRANCH}" \
            --github.username="${GITHUB_USER}" \
            --github.token="${GITHUB_TOKEN}" \
            --polaris.project.name="${PROJECT}" \
            --polaris.branch.name="${BRANCH}" \
            --polaris.url="${_POLARIS_SERVER_URL}" \
            --polaris.token="${POLARIS_ACCESS_TOKEN}" \
            --blackduck.project.name="${PROJECT}:${VERSION}" \
            --blackduck.url="${_BLACKDUCK_SERVER_URL}" \
            --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
            --jira.enable="false" \
            --codedx.url="${_CODEDX_SERVER_URL}/codedx" \
            --codedx.api.key="${CODEDX_TOKEN}" \
            --codedx.project.id="$projectID" \
            --IS_SAST_ENABLED="false" \
            --IS_SCA_ENABLED="false" \
            --IS_DAST_ENABLED="false"
        '''
        script {
          env.IS_SAST_ENABLED = sh(script:'jq -r ".security.activities.sast.enabled" result.json', returnStdout: true).trim()
          env.IS_SCA_ENABLED = sh(script:'jq -r ".security.activities.sca.enabled" result.json', returnStdout: true).trim()
          env.IS_DAST_ENABLED = sh(script:'jq -r ".security.activities.dast.enabled" result.json', returnStdout: true).trim()
          env.IS_DASTPLUSM_ENABLED = sh(script:'jq -r ".security.activities.dastplusm.enabled" result.json', returnStdout: true).trim()
          env.IS_SASTPLUSM_ENABLED = sh(script:'jq -r ".security.activities.sastplusm.enabled" result.json', returnStdout: true).trim()
          env.IS_IMAGESCAN_ENABLED = sh(script:'jq -r ".security.activities.imagescan.enabled" result.json', returnStdout: true).trim()
        }
      }
    }

    stage('SAST - Coverity on Polaris') {
      when {
        expression { env.IS_SAST_ENABLED == "true" }
      }
      steps {
        sh '''
          #if [ ${env.IS_SAST_ENABLED} = "true" ]; then
          echo "Running Coverity on Polaris"
          rm -fr /tmp/polaris 2>/dev/null
          wget -q ${_POLARIS_SERVER_URL}/api/tools/polaris_cli-linux64.zip
          unzip -j polaris_cli-linux64.zip -d /tmp
          rm polaris_cli-linux64.zip
          /tmp/polaris --persist-config --co project.name="${PROJECT}" --co project.branch="${BRANCH}" --co capture.build.buildCommands="null" --co capture.build.cleanCommands="null" --co capture.fileSystem="null" --co capture.coverity.autoCapture="enable" configure
          /tmp/polaris analyze -w
          #else
          #  echo "Skipping Coverity on Polaris based on prescription"
          #fi
        '''
      }
    }
    stage ('SCA - Black Duck') {
      when {
        expression { env.IS_SCA_ENABLED == "true" }
      }
      steps {
        sh '''
          echo "Running BlackDuck"
          rm -fr /tmp/detect7.sh
          curl -s -L https://detect.synopsys.com/detect7.sh > /tmp/detect7.sh
          bash /tmp/detect7.sh --blackduck.url="${_BLACKDUCK_SERVER_URL}" --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" --detect.project.name="${PROJECT}" --detect.project.version.name="${VERSION}" --blackduck.trust.cert=true
        '''
      }
    }

    stage ('IAST - Seeker') {
      when {
        expression { env.IS_DAST_ENABLED == "true" }
      }
      steps {
        sh '''#!/bin/bash
          if [ ! -z ${SERVER_WORKINGDIR} ]; then cd ${SERVER_WORKINGDIR}; fi

          sh -c "$( curl -k -X GET -fsSL --header 'Accept: application/x-sh' \"${_SEEKER_SERVER_URL}/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&projectKey=${SEEKER_PROJECT_KEY}&webServer=TOMCAT&flavor=DEFAULT&agentName=&accessToken=\")"

          export SEEKER_PROJECT_VERSION=${VERSION}
          export SEEKER_AGENT_NAME=${AGENT}
          export MAVEN_OPTS=-javaagent:seeker/seeker-agent.jar

          serverMessage=$(/tmp/serverStart.sh --startCmd="${SERVER_START}" --startedString="${SERVER_STRING}" --project="${PROJECT}" --timeout="60s" &)
          if [[ $serverMessage == ?(-)+([0-9]) ]]; then #Check if value passed back is numeric (PID) or string (Error message).
            echo "Running IAST Tests"

            testRunID=$(curl -X 'POST' "${_SEEKER_SERVER_URL}/rest/api/latest/testruns" -H 'accept: application/json' -H 'Content-Type: application/x-www-form-urlencoded' -H "Authorization: ${SEEKER_TOKEN}" -d "type=AUTO_TRIAGE&statusKey=FIXED&projectKey=${SEEKER_PROJECT_KEY}" | jq -r ".[]".key)
            echo "Run ID is : "$testRunID
            npm test

            selenium-side-runner -c "browserName=firefox moz:firefoxOptions.args=[-headless]" --output-directory=/tmp ${WORKSPACE}/selenium/jHipster.side

            # Give Seeker some time to do it's stuff; API collation, testing etc.
            sleep ${SEEKER_RUN_TIME}

            testResponse=$(curl -X 'PUT' "${_SEEKER_SERVER_URL}/rest/api/latest/testruns/$testRunID/close" -H 'accept: application/json' -H 'Content-Type: application/x-www-form-urlencoded' -H "Authorization: ${SEEKER_TOKEN}" -d 'completed=true')
            echo "Finished Testing. [$testResponse]"

            kill $serverMessage
          else
            echo $serverMessage
            return 1
          fi
        '''
      }
    }

    stage ('IO Workflow - Code Dx') {
      steps {
        sh '''
          projectID=$(/tmp/getProjectID.sh --url=${_CODEDX_SERVER_URL} --apikey=${CODEDX_TOKEN} --project=${PROJECT})
          /tmp/prescription.sh \
            --stage="WORKFLOW" \
            --persona="devsecops" \
            --project.name="${PROJECT}" \
            --io.url="${_IO_SERVER_URL}/api/ioiq/" \
            --io.token="${IO_TOKEN}" \
            --manifest.type="json" \
            --asset.id="${PROJECT}" \
            --workflow.url="${_IO_SERVER_URL}/api/workflowengine/" \
            --workflow.version="2022.4.1" \
            --file.change.threshold="10" \
            --sast.rescan.threshold="20" \
            --sca.rescan.threshold="20" \
            --scm.type="github" \
            --scm.owner="${GITHUB_USER}" \
            --scm.repo.name="${GITHUB_PROJECT}" \
            --scm.branch.name="${BRANCH}" \
            --github.username="${GITHUB_USER}" \
            --github.token="${GITHUB_TOKEN}" \
            --polaris.project.name="${PROJECT}" \
            --polaris.branch.name="${BRANCH}" \
            --polaris.url="${_POLARIS_SERVER_URL}" \
            --polaris.token="${POLARIS_ACCESS_TOKEN}" \
            --blackduck.project.name="${PROJECT}:${VERSION}" \
            --blackduck.url="${_BLACKDUCK_SERVER_URL}" \
            --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
            --jira.enable="false" \
            --codedx.url="${_CODEDX_SERVER_URL}/codedx" \
            --codedx.api.key="${CODEDX_TOKEN}" \
            --codedx.project.id="$projectID" \
            --IS_SAST_ENABLED="${IS_SAST_ENABLED}" \
            --IS_SCA_ENABLED="${IS_SCA_ENABLED}" \
            --IS_DAST_ENABLED="${IS_DAST_ENABLED}"

          java -jar WorkflowClient.jar --workflowengine.url="${_IO_SERVER_URL}/api/workflowengine/" --io.manifest.path=synopsys-io.json
        '''
      }
    }
    stage('Clean Workspace') {
      steps {
        cleanWs()
      }
    }
  }
}
