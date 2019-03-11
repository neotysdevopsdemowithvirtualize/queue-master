

pipeline {
    agent  { label 'master' }
    tools {
        maven 'Maven 3.6.0'
        jdk 'jdk8'
    }
  environment {
    VERSION="0.1"
    APP_NAME = "queue-master"
    TAG = "neotysdevopsdemo/${APP_NAME}"
    TAG_DEV = "${TAG}:DEV-${VERSION}"
    TAG_STAGING = "${TAG}-stagging:${VERSION}"
    NL_DT_TAG="app:${env.APP_NAME},environment:dev"
    QUEUEMASTER_ANOMALIEFILE="$WORKSPACE/monspec/queue-master_anomalieDection.json"
    DYNATRACEID="${env.DT_ACCOUNTID}.live.dynatrace.com"
    DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
    NLAPIKEY="${env.NL_WEB_API_KEY}"
    OUTPUTSANITYCHECK="$WORKSPACE/infrastructure/sanitycheck.json"
    DYNATRACEPLUGINPATH="$WORKSPACE/lib/DynatraceIntegration-3.0.1-SNAPSHOT.jar"
    BASICCHECKURI="/health"
    GROUP = "neotysdevopsdemo"
    COMMIT = "DEV-${VERSION}"

  }
  stages {
      stage('Checkout') {
          agent { label 'master' }
          steps {
              git  url:"https://github.com/${GROUP}/${APP_NAME}.git",
                      branch :'master'
          }
      }
    stage('Maven build') {
      steps {

            sh "mvn -B clean package -DdynatraceId=$DYNATRACEID -DneoLoadWebAPIKey=$NLAPIKEY -DdynatraceApiKey=$DYNATRACEAPIKEY -Dtags=${NL_DT_TAG} -DoutPutReferenceFile=$OUTPUTSANITYCHECK -DcustomActionPath=$DYNATRACEPLUGINPATH -DjsonAnomalieDetectionFile=$QUEUEMASTER_ANOMALIEFILE"

      }

    }
    stage('Docker build') {

        steps {
            withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                sh "cp ./target/*.jar ./docker/${APP_NAME}"
                sh "docker build -t ${TAG_DEV} $WORKSPACE/docker/${APP_NAME}/"
                sh "docker login --username=${USER} --password=${TOKEN}"
                sh "docker push ${TAG_DEV}"
            }

        }
    }


    stage('Deploy to dev namespace') {

      steps {

          sh "sed -i 's,TAG_TO_REPLACE,${TAG_DEV},' $WORKSPACE/docker-compose.yml"
          sh 'docker-compose -f $WORKSPACE/docker-compose.yml up -d'

      }
    }
   /* stage('DT Deploy Event') {
        when {
            expression {
            return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
            }
        }
        steps {
          container("curl") {
            // send custom deployment event to Dynatrace
            sh "curl -X POST \"$DT_TENANT_URL/api/v1/events?Api-Token=$DT_API_TOKEN\" -H \"accept: application/json\" -H \"Content-Type: application/json\" -d \"{ \\\"eventType\\\": \\\"CUSTOM_DEPLOYMENT\\\", \\\"attachRules\\\": { \\\"tagRule\\\" : [{ \\\"meTypes\\\" : [\\\"SERVICE\\\"], \\\"tags\\\" : [ { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"app\\\", \\\"value\\\" : \\\"${env.APP_NAME}\\\" }, { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"environment\\\", \\\"value\\\" : \\\"dev\\\" } ] }] }, \\\"deploymentName\\\":\\\"${env.JOB_NAME}\\\", \\\"deploymentVersion\\\":\\\"${_VERSION}\\\", \\\"deploymentProject\\\":\\\"\\\", \\\"ciBackLink\\\":\\\"${env.BUILD_URL}\\\", \\\"source\\\":\\\"Jenkins\\\", \\\"customProperties\\\": { \\\"Jenkins Build Number\\\": \\\"${env.BUILD_ID}\\\",  \\\"Git commit\\\": \\\"${env.GIT_COMMIT}\\\" } }\" "
          }
        }
    }*/
    stage('Start NeoLoad infrastructure') {
        steps {
            sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml up -d'

        }

    }
    stage('Join Load Generators to Application') {
        steps {
            sh 'docker network connect queue-master_master_default docker-lg1'
        }
    }
    stage('Run health check in dev') {
        agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=queue-master_master_default'
                dir 'infrastructure/infrastructure/neoload/controller'
            }
        }
      steps {
        echo "Waiting for the service to start..."
        sleep 300

          script {
              neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                      project: "$WORKSPACE/target/neoload/queuemaster_NeoLoad/queuemaster_NeoLoad.nlp",
                      testName: 'HealthCheck_queuemaster_${VERSION}_${BUILD_NUMBER}',
                      testDescription: 'HealthCheck_queuemaster_${VERSION}_${BUILD_NUMBER}',
                      commandLineOption: "-nlweb -loadGenerators $WORKSPACE/infrastructure/infrastructure/neoload/lg/lg.yaml -nlwebToken $NLAPIKEY -variables host=${env.APP_NAME},port=80,basicPath=${BASICCHECKURI}",
                      scenario: 'DynatraceSanityCheck', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                      trendGraphs: [
                              [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                              'ErrorRate'
                      ]
          }




      }
    }
    stage('Sanity Check') {
        agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=queue-master_master_default'
                dir 'infrastructure/infrastructure/neoload/controller'
            }
        }
              steps {
                  script {
                      neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                              project: "$WORKSPACE/target/neoload/queuemaster_NeoLoad/queuemaster_NeoLoad.nlp",
                              testName: 'DynatraceSanityCheck_queuemaster_${VERSION}_${BUILD_NUMBER}',
                              testDescription: 'DynatraceSanityCheck_queuemaster_${VERSION}_${BUILD_NUMBER}',
                              commandLineOption: "-nlweb -loadGenerators $WORKSPACE/infrastructure/infrastructure/neoload/lg/lg.yaml -nlwebToken $NLAPIKEY -variables host=${env.APP_NAME},port=80",
                              scenario: 'DYNATRACE_SANITYCHECK', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                              trendGraphs: [
                                      [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                                      'ErrorRate'
                              ]
                  }


                  echo "push ${OUTPUTSANITYCHECK}"
                  withCredentials([usernamePassword(credentialsId: 'git-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                      sh "git config --global user.email ${env.GITHUB_USER_EMAIL}"
                      sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/carts"
                      sh "git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/*"
                      sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/carts"
                    //  sh "git add ${OUTPUTSANITYCHECK}"
                     // sh "git commit -m 'Update Sanity_Check_${BUILD_NUMBER} ${env.APP_NAME} '"
                      //  sh "git pull -r origin master"
                      //#TODO handle this exeption
                      //   sh "git push origin HEAD:master"

                  }

              }
    }
    stage('Run functional check in dev') {
        agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=queue-master_master_default'
                dir 'infrastructure/infrastructure/neoload/controller'
            }
        }

      steps {
          script {
              neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                      project: "$WORKSPACE/target/neoload/queuemaster_NeoLoad/queuemaster_NeoLoad.nlp",
                      testName: 'FuncCheck_queuemaster__${VERSION}_${BUILD_NUMBER}',
                      testDescription: 'FuncCheck_queuemaster__${VERSION}_${BUILD_NUMBER}',
                      commandLineOption: "-nlweb -loadGenerators $WORKSPACE/infrastructure/infrastructure/neoload/lg/lg.yaml -nlwebToken $NLAPIKEY -variables host=${env.APP_NAME},port=80,basicPath=${BASICCHECKURI}",
                      scenario: 'QueueMaster_Load', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                      trendGraphs: [
                              [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                              'ErrorRate'
                      ]
          }

      }
    }
    stage('Mark artifact for staging namespace') {
        steps {
            withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                sh "docker login --username=${USER} --password=${TOKEN}"
                sh "docker tag ${TAG_DEV} ${TAG_STAGING}"
                sh "docker push ${TAG_STAGING}"
            }
        }
    }

  }
  post {
          always {

              sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/doker-compose.yml down'
              sh 'docker-compose -f $WORKSPACE/docker-compose.yml down'

          }

        }
}
