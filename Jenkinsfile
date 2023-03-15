def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]

pipeline{

     agent any

     tools{
        maven "MAVEN3"
        jdk "OracleJDK8"
     }
      
     environment {
        SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = 'admin'
        NEXUS_PASS = "admin"
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUSIP = '172.31.24.73'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vprofile-maven-group'
        NEXUS_LOGIN = 'nexus_login'
        SONARSERVER = "sonarserver"
        SONARSCANNER = "sonarscanner"



     }

     stages {
        
        stage ("Build"){
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }

            post {
                success {
                    echo "Now archiving."
                    archiveArtifacts artifacts: "**/*.war"
                }
            }

 
        }
        stage('Test') {
           steps {
            sh "mvn -s settings.xml test"
           }
        }

        stage ("Checkstyle Analysis"){
             steps{
                sh "mvn -s settings.xml checkstyle:checkstyle"
             }
        }

        stage ("Sonar Analysis") {

            environment {
                scannerHome = tool "${SONARSCANNER}"

            }

            steps {
                 
                withSonarQubeEnv ("${SONARSERVER}"){
                     sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }

        }

      stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage ("UploadArtifact"){
            steps {
                 nexusArtifactUploader(
                  nexusVersion: 'nexus3',
                  protocol: 'http',
                  nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                  groupId: 'QA',
                  version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                  repository: "${RELEASE_REPO}",
                  credentialsId: "${NEXUS_LOGIN}",
                  artifacts: [
                    [artifactId: 'vproapp',
                     classifier: '',
                     file: 'target/vprofile-v2.war',
                     type: 'war']
                  ]
                )
            }
        }

        stage('Ansible Deploy to staging') {
            steps {
                ansiblePlaybook( [
                playbook: 'ansible/site.yaml',
                inventory: 'ansible/stage.inventory',
                credentialsId: 'applogin',
                installation: 'ansible'
                disableHostKeyChecking: true,
                extraVars: [
                  USER: "admin"
                  PASS: "admin"
                  nexusip: "172.31.18.143"
                  reponame: "vprofile-release"
                  groupid: "QA"
                  time: "${env.BUILD_TIMESTAMP}"
                  build: "${env.BUILD_ID}"
                  artifactid: "vproapp"
                  vprofile_version: "vproapp-${env.BUILD_ID}-${env.BUILD_TIMESTAMP}.war"
                ]
                colorized: true)
                ]
            }
        }


     }
     post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#isreal-channel',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }

}
