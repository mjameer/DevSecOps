def appName=shopping-cart

// SonarQube Configurations
def sonarQubePath = '/bin/sonar-scanner'
def sonarHost = 'https://sonarqube.com'
def sonarSettings = "sonar.properties"

//Artifactory Configuration
def pattern = "/apps/jenkins-agent/workspace/${appName}/target/*.war"
def target = "libs-snapshot-local/${appName}/"

pipeline {
    agent any
    tools{
        jdk  'jdk11'
        maven  'maven3'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', changelog: false, credentialsId: 'credentialsIdInJenkins', poll: false, url: 'https://github.com/mjameer/DevSecOps.git'
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn clean package -DskipTests=true"
            }
        }
        
        stage('OWASP Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ ', odcInstallation: 'DP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        // Nexus Plugin must be installed and in Nexus dashboard app shopping-cart must be configured
        stage('Nexus IQ Evaluation') {
                try {
                    def evalResult = nexusPolicyEvaluation failBuildOnNetworkError: false, iqApplication: "${appName}", iqScanPatterns: [
                        [scanPattern: '**/*.jar'],
                        [scanPattern: '**/*.war'],
                        [scanPattern: '**/*.ear'],
                        [scanPattern: '**/*.zip'],
                        [scanPattern: '**/*.js']
                    ], iqStage: 'build', jobCredentialsId: ''
                } catch (Exception err) {
                    println(err.toString());
                    println(err.getMessage());
                    println(err.getStackTrace());
                    currentBuild.result = 'ERROR'
                }

                currentBuild.result = 'SUCCESS'
            }

        stage('Sonar Analysis') {
            withSonarQubeEnv('sonarqube') {
            sh "${sonarQubePath} -e -Dsonar.host.url=${sonarHost} '-Dproject.settings=${sonarSettings}'"
            }

            // Added Quality gate to check new code is better than old code
            def qualityGate = waitForQualityGate() // Wait for SonarQube analysis results
            if (qualityGate.status != 'OK') {
                error("SonarQube Quality Gate check failed!")
            }
        }

        stage('White-hat SAST Evaluation') {

            withCredentials([usernamePassword(credentialsId: 'devops-artifactory-automation-account', passwordVariable: 'ARTIFACTORY_PASSWORD', usernameVariable: 'ARTIFACTORY_USER')]) {

                projectName = "${appName}"
                artifactName = projectName + "-whitehat"
                artifactLocation = 'https://artifactory.com/artifactory/' + 'local-whitehat' + '/' + projectName + '/'
                artifactURL = artifactLocation + artifactName
                antFileCmd = "echo '<project basedir=\".\" default=\"makeZip\" name=\"Workspace Zip\">' > antFile\n" +
                    "echo '<target description=\"Create a zip for the workspace\" name=\"makeZip\">' >> antFile\n" +
                    "echo '<tstamp>' >> antFile\n" +
                    "echo '<format property=\"BUILD_TIME_STAMP\" pattern=\"yyyyMMddHHmm\"/>' >> antFile\n" +
                    "echo '</tstamp>' >> antFile\n" +
                    "echo '<property name=\"basefolderName\" value=\"" + artifactName + "\"/>' >> antFile\n" +
                    "echo '<tar destfile=\"" + artifactName + ".tar\">' >> antFile\n" +
                    "echo '<tarfileset dir=\".\" preserveLeadingSlashes=\"false\" excludes=\"**/*.jpg,**/*.txt,**/*.doc,**/*.docx,**/*.gif,**/*.png,**/*.log\" includes=\"*,**/*\">' >> antFile\n" +
                    "echo '</tarfileset>' >> antFile\n" +
                    "echo '</tar>' >> antFile\n" +
                    "echo '<gzip destfile=\"" + artifactName + ".tar.gz\" src=\"" + artifactName + ".tar\"/>'  >> antFile\n" +
                    "echo '<delete file=\"" + artifactName + ".tar\" quiet=\"true\"/>' >> antFile\n" +
                    "echo '<move file=\"" + artifactName + ".tar.gz\" todir=\"WhiteHat\"/>' >> antFile\n" +
                    "echo '</target>' >> antFile\n" +
                    "echo '</project>' >> antFile\n"
                echo "########## AntFile Command " + antFileCmd
                sh 'cd $WORKSPACE\n' +
                    antFileCmd +
                    'cat antFile\n' +
                    'ant -f antFile\n' +
                    'cd WhiteHat;ls\n' +
                    'curl -u $ARTIFACTORY_USER:$ARTIFACTORY_PASSWORD -X PUT "' + artifactLocation + '" -T ' + artifactName + '.tar.gz\n' +
                    'RESULT=$?\n' +
                    'if [ $RESULT -ne 0 ]; then\n' +
                    'echo "Unable to upload WhiteHat file ' + artifactName + '.tar.gz to Artifactory."\n' +
                    'exit -1\n' +
                    'fi\n' +
                    'cd $WORKSPACE\n' +
                    'rm -rf WhiteHat || true\n'
            }

        }
        
        stage('Docker Build & Push') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'ID_CONFIGURED_IN_JENKINS', toolName: 'docker') {
                        
                        sh "docker build -t ${appName} -f Dockerfile ."
                        sh "docker tag  ${appName} mjameer/${appName}:latest"
                        sh "docker push mjameer/${appName}:latest"
                    }
                }
            }
        }

        stage('Upload Artifact to Artifactory') {
        // Define default timeout for job to wait before it automatically aborts
        timeout(time: 1, unit: 'DAYS') {
            // This must approved only by Onshore Primary
            input message: 'App Team - Approve Publish to Artifactory?', submitter: "${submitter}"

            // Get this input after above Approval step
            def userInput = input(
                id: 'userInput', message: 'App Team - Approve Publish to Artifactory ?',
                parameters: [
                        string(defaultValue: 'Change Number ?',
                                description: 'Change Number',
                                name: 'changeNo'),
                        string(defaultValue: 'Implemented By ?',
                                description: 'Change Implemented By',
                                name: 'changeImpBy'),
                ])

                // Save to variables. Default to empty string if not found.
            changeNo = userInput.changeNo?:''
            changeImpBy = userInput.changeImpBy?:''
                    // Echo to console
                    echo("Change Number: ${changeNo}")
                    echo("Change Implemented By: ${changeImpBy}")

                     def server = Artifactory.server('bluemix')
                     def buildInfo = Artifactory.newBuildInfo()
                     buildInfo.env.capture = true
                     buildInfo.retention maxBuilds: 10
                     def uploadSpec = """{
                       "files": [
                            {
                              "pattern": "${pattern}",
                              "target": "${target}-${changeNo}${extension}",
                              "props": "build.name=${appName};build.number=${changeNo}",
                              "recursive": "true",
                              "flat":"false"
                            }
                       ]
                    }"""
                     server.upload(uploadSpec)
                     server.publishBuildInfo(buildInfo)
        }
        }
}
