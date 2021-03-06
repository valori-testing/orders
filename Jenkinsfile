
pipeline {
agent  { label 'master' }
    tools {
        maven 'Maven 3.6.0'
        jdk 'jdk8'
    }
    environment {
    VERSION="0.1"
    APP_NAME = "orders"
    TAG = "neotysdevopsdemo/${APP_NAME}"
    TAG_DEV = "${TAG}:DEV-${VERSION}"
    NL_DT_TAG = "app:${env.APP_NAME},environment:dev"
    CARTS_ANOMALIEFILE="$WORKSPACE/monspec/orders_anomalieDection.json"
    TAG_STAGING = "${TAG}-stagging:${VERSION}"
    DYNATRACEID="${env.DT_ACCOUNTID}.live.dynatrace.com"
    DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
    NLAPIKEY="${env.NL_WEB_API_KEY}"
    OUTPUTSANITYCHECK="$WORKSPACE/infrastructure/sanitycheck.json"
    DYNATRACEPLUGINPATH="$WORKSPACE/lib/DynatraceIntegration-3.0.1-SNAPSHOT.jar"
    GROUP = "neotysdevopsdemo"
    COMMIT = "DEV-${VERSION}"
    BASICCHECKURI="/health"
    ORDERSURI="/orders"
  }
  stages {
   stage('Checkout') {

              steps {
                  git  url:"https://github.com/${GROUP}/${APP_NAME}.git",
                          branch :'master'
              }
          }
    stage('Maven build') {
                steps {
          sh "mvn -B clean package -DdynatraceId=$DYNATRACEID -DneoLoadWebAPIKey=$NLAPIKEY -DdynatraceApiKey=$DYNATRACEAPIKEY -Dtags=${NL_DT_TAG} -DoutPutReferenceFile=$OUTPUTSANITYCHECK -DcustomActionPath=$DYNATRACEPLUGINPATH -DjsonAnomalieDetectionFile=$CARTS_ANOMALIEFILE"

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

    stage('Deploy to dev ') {

        steps {
            sh "sed -i 's,TAG_TO_REPLACE,${TAG_DEV},' $WORKSPACE/docker-compose.yml"
            sh 'docker-compose -f $WORKSPACE/docker-compose.yml up -d'

        }
    }
    /*stage('DT Deploy Event') {
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
                sh 'docker network connect orders_master_default docker-lg1'
            }
        }
    stage('Run health check in dev') {
        agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=orders_master_default'
                dir 'infrastructure/infrastructure/neoload/controller'
                  reuseNode true
            }
        }
      steps {
          echo "Waiting for the service to start..."
          sleep 90

          script {
              neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                      project: "$WORKSPACE/target/neoload/Orders_NeoLoad/Orders_NeoLoad.nlp",
                      testName: 'HealthCheck_orders_${VERSION}_${BUILD_NUMBER}',
                      testDescription: 'HealthCheck_orders_${VERSION}_${BUILD_NUMBER}',
                      commandLineOption: "-nlweb -L Population_BasicCheckTesting=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables host=orders,port=80,basicPath=${BASICCHECKURI}",
                      scenario: 'DynatraceSanityCheck', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                      trendGraphs: [
                              [name: 'Limit test HealthAPI Response time', curve: ['BasicCheckTesting>Actions>BasicCheck'], statistic: 'average'],
                              'ErrorRate'
                      ]
          }
      }

    }
     stage('Sanity Check') {
         agent {
             dockerfile {
                 args '--user root -v /tmp:/tmp --network=orders_master_default'
                 dir 'infrastructure/infrastructure/neoload/controller'
                 reuseNode true
             }
         }
          steps {
             script {
                  neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                          project: "$WORKSPACE/target/neoload/Orders_NeoLoad/Orders_NeoLoad.nlp",
                          testName: 'DynatraceSanityCheck_orders_${VERSION}_${BUILD_NUMBER}',
                          testDescription: 'DynatraceSanityCheck_orders_${VERSION}_${BUILD_NUMBER}',
                          commandLineOption: "-nlweb -L  Population_Dynatrace_SanityCheck=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables host=orders,port=80",
                          scenario: 'DYNATRACE_SANITYCHECK', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                          trendGraphs: [
                                  'ErrorRate'
                          ]
              }



               echo "push ${OUTPUTSANITYCHECK}"
              withCredentials([usernamePassword(credentialsId: 'git-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                  sh "git config --global user.email ${env.GITHUB_USER_EMAIL}"
                  sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/carts"
                  sh "git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/*"
                  sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/carts"
                  //sh "git add ${OUTPUTSANITYCHECK}"
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
                args '--user root -v /tmp:/tmp --network=orders_master_default'
                dir 'infrastructure/infrastructure/neoload/controller'
                reuseNode true
            }
        }

      steps {
          script {
              neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                      project: "$WORKSPACE/target/neoload/Orders_NeoLoad/Orders_NeoLoad.nlp",
                      testName: 'FuncCheck_orders__${VERSION}_${BUILD_NUMBER}',
                      testDescription: 'FuncCheck_orders__${VERSION}_${BUILD_NUMBER}',
                      commandLineOption: "-nlweb Population_Orders=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables carts_host=orders,carts_port=80,orderPath=${ORDERSURI}",
                      scenario: 'Order_Load', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                      trendGraphs: [
                              [name: 'Limit test Orders API Response time', curve: ['Orders>Actions>Orders'], statistic: 'average'],
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

            sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml down'
            sh 'docker-compose -f $WORKSPACE/docker-compose.yml down'
            cleanWs()
            sh 'docker volume prune'
        }

    }
}
