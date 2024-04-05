pipeline {


  environment {
    devRegistry = 'ghcr.io/datakaveri/onboarding-server-dev'
    deplRegistry = 'ghcr.io/datakaveri/onboarding-server-depl'
    registryUri = 'https://ghcr.io'
    registryCredential = 'datakaveri-ghcr'
    GIT_HASH = GIT_COMMIT.take(7)
  }


  agent {
    node {
      label 'slave1'
    }
  }


  stages {
    stage('Build images') {
      steps{
        script {
          devImage = docker.build( devRegistry, "-f ./docker/dev.dockerfile .")
          deplImage = docker.build( deplRegistry, "-f ./docker/depl.dockerfile .")
        }
      }
    }
    stage('Unit Tests and Code Coverage Test'){
      steps{
        script{
          sh 'cp /home/ubuntu/configs/5.5.0/onboarding-config-test.json ./secrets/all-verticles-configs/config-test.json'
           catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
              sh "mvn clean test checkstyle:checkstyle pmd:pmd"
            }  
        }
      }
      post{
        always {
          xunit (
          thresholds: [ skipped(failureThreshold: '0'), failed(failureThreshold: '0') ],
          tools: [ JUnit(pattern: 'target/surefire-reports/*.xml') ]
        )
        jacoco classPattern: 'target/classes', execPattern: 'target/jacoco.exec', sourcePattern: 'src/main/java', exclusionPattern:'**/*VertxEBProxy.class,**/Constants.class,**/*VertxProxyHandler.class,**/*Verticle.class,**/*Service.class,iudx/onboarding/server/deploy/*.class'
          recordIssues(
            enabledForFailure: true,
            blameDisabled: true,
            forensicsDisabled: true,
            qualityGates: [[threshold:0, type: 'TOTAL', unstable: false]],
            tool: checkStyle(pattern: 'target/checkstyle-result.xml')
          )
          recordIssues(
            enabledForFailure: true,
            blameDisabled: true,
            forensicsDisabled: true,
            qualityGates: [[threshold:25, type: 'TOTAL', unstable: false]],
            tool: pmdParser(pattern: 'target/pmd.xml')
          )
      }
        failure{
          error "Test failure. Stopping pipeline execution!"
        }
        cleanup{
          script{
            sh 'sudo rm -rf target/'
          }
        }        
      }
    }
    stage('Start onboarding-server for Performance/Integration Testing'){
      steps{
        script{
          sh 'scp src/test/resources/iudx-onboarding-server-test-suite.postman_collection.json jenkins@jenkins-master:/var/lib/jenkins/iudx/onboarding/Newman/'
          sh 'docker compose -f docker-compose.test.yml up -d integTest'
          sh 'sleep 45'
        }
      }
      post{
        failure{
          script{
            sh 'docker compose -f docker-compose.test.yml down --remove-orphans'
          }
        }
      }
    }

    stage('Integration Tests and OWASP ZAP pen test'){
      steps{
        node('built-in') {
          script{
            startZap ([host: '0.0.0.0', port: 8090, zapHome: '/var/lib/jenkins/tools/com.cloudbees.jenkins.plugins.customtools.CustomTool/OWASP_ZAP/ZAP_2.11.0'])
            sh 'curl http://0.0.0.0:8090/JSON/pscan/action/disableScanners/?ids=10096'
          }
        }
        script{
            sh 'mkdir configs'
            sh 'cp /home/ubuntu/configs/5.5.0/onboarding-config-test.json ./configs/config-test.json'
            sh 'mvn test-compile failsafe:integration-test -DskipUnitTests=true -DintTestProxyHost=jenkins-master-priv -DintTestProxyPort=8090 -DintTestHost=jenkins-slave1 -DintTestPort=8080'
        }
        node('built-in') {
          script{
            runZapAttack()
          }
        }
      }
      post{
        always{
           xunit (
             thresholds: [ skipped(failureThreshold: '0'), failed(failureThreshold: '0') ],
             tools: [ JUnit(pattern: 'target/failsafe-reports/*.xml') ]
             )
          node('built-in') {
            script{
              archiveZap failHighAlerts: 1, failMediumAlerts: 1, failLowAlerts: 1
            }
          }
        }
        failure{
          script{
            sh 'rm -rf configs'
          }
          error "Test failure. Stopping pipeline execution!"
        }
        cleanup{
          script{
            sh 'docker compose -f docker-compose.test.yml down --remove-orphans'
          }
        }
      }
    }

    stage('Push Images') {
      when {
        allOf {
          anyOf {
            changeset "docker/**"
            changeset "docs/**"
            changeset "pom.xml"
            changeset "src/main/**"
            triggeredBy cause: 'UserIdCause'
          }
          expression {
            return env.GIT_BRANCH == 'origin/1.0.1';
          }
        }
  	  }
      steps {
        script {
          docker.withRegistry( registryUri, registryCredential ) {
            devImage.push("1.0.1-${env.GIT_HASH}")
            deplImage.push("1.0.1-${env.GIT_HASH}")
          }
        }
      }
    }


  }
  post{
    failure{
      script{
        sh 'rm -rf configs'
        if (env.GIT_BRANCH == 'origin/1.0.1')
        emailext recipientProviders: [buildUser(), developers()], to: '$ONBOARDING_RECIPENTS, $DEFAULT_RECIPIENTS', subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!', body: '''$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:
Check console output at $BUILD_URL to view the results.'''
      }
    }
  }
}

