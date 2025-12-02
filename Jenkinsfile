def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline {

    agent any

    tools {
        maven "MAVEN3.9"
        jdk "JDK21"
    }

    environment {
        SNAP_REPO      = 'vprofile-snapshot'
        NEXUS_USER     = 'admin'
        NEXUS_PASS     = 'Thanks@#12'
        RELEASE_REPO   = 'vprofile-repo'
        CENTRAL_REPO   = 'vpro-maven-central'
        NEXUSIP        = '172.31.2.175'
        NEXUSPORT      = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN    = 'nexuslogin'
        SONARSERVER    = 'sonarserver'
        SONARSCANNER   = 'sonar7.3'
        NEXUSPASS      = credentials('nexuslogin')

        // FIX: Proper timestamp for Nexus + Ansible
        BUILD_TIME = "${new Date().format('yyyyMMdd-HHmmss')}"
    }

    stages {

        stage('Build') {
            steps {
                sh 'mvn -DskipTests install'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {
            tools {
                jdk "JDK11"
            }
            environment {
                scannerHome = tool "${SONARSCANNER}"
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    sh '''
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Upload Artifact to Nexus") {
            steps {
                script {
                    def ARTIFACT_VERSION = "${env.BUILD_ID}-${env.BUILD_TIME}"

                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                        groupId: 'QA',
                        version: ARTIFACT_VERSION,
                        repository: "${RELEASE_REPO}",
                        credentialsId: "${NEXUS_LOGIN}",
                        artifacts: [[
                            artifactId: 'vproapp',
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war'
                        ]]
                    )

                    // Export final version for Ansible
                    env.FINAL_WAR = "vproapp-${ARTIFACT_VERSION}.war"
                    env.ARTIFACT_FOLDER = ARTIFACT_VERSION
                }
            }
        }

        stage('Ansible Deploy to Staging') {
            steps {
                ansiblePlaybook([
                    inventory   : 'ansible/stage.inventory',
                    playbook    : 'ansible/site.yml',
                    installation: 'ansible',
                    colorized   : true,
                    credentialsId: 'applogin',
                    disableHostKeyChecking: true,

                    extraVars: [
                        USER: "admin",
                        PASS: "${NEXUSPASS}",
                        nexusip: "${NEXUSIP}",
                        reponame: "${RELEASE_REPO}",
                        groupid: "QA",

                        // FIXED — PERFECT MATCH FOR NEXUS STRUCTURE
                        artifactid: "vproapp",
                        build: "${env.ARTIFACT_FOLDER}",
                        vprofile_version: "${env.FINAL_WAR}"
                    ]
                ])
            }
        }

    }

    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#devopscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\nMore info: ${env.BUILD_URL}"
        }
    }
}