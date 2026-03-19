def notify(name, id, status, test_res) {
    echo("sending email notification...")
    echo("Status for job ${name} with id ${id} is ${status}")
    echo("Test results: ${test_res}")
}
def TEST_OUT = ""
pipeline{
    agent any
    
    parameters {
      string defaultValue: 'pipeline', name: 'ARTIFACT_NAME', trim: true
      booleanParam 'FAIL_PIPELINE'
      booleanParam defaultValue: true, name: 'RUN_TEST'
      string defaultValue: 'main', name: "GIT_BRANCH", trim: true
      booleanParam defaultValue: true, name: 'SEND_EMAIL'
    }

    stages {
        stage('Download') {
            environment {
                TEST_OUTPUT_ENV = ""
            }
            steps {
                withCredentials(
                    [usernamePassword(credentialsId: 'git-pass', passwordVariable: 'pass', usernameVariable: 'usr')]
                ) {
                    bat 'echo "Credentials are: %usr% / %pass%" >> creds.txt'
                }
                script{
                    if ( params.FAIL_PIPELINE == true ){
                       error("Pipeline was doomed to fail")
                    }
                }
                cleanWs()
                echo (message: "Downloading from git...")
                rtDownload (
                    serverId: 'jfrog_artifactory',
                    spec: '''{
                          "files": [
                           {
                               "pattern": "generic-local/libraries/printer.zip",
                               "target": "./",
                               "flat": "true"
                           }
                          ]
                    }'''
                )
                
                dir('pipeline') {
                    git branch: "${params.GIT_BRANCH}", url: 'https://github.com/KLevon/jenkins-course'
                }
                unzip (
                    zipFile: "printer.zip",
                    dir: "pipeline"
                )
            }
        }
        stage('Build'){
            steps {
                echo (message: "Building...")
                dir('pipeline'){
                    bat (
                        script: """
                            Makefile.bat
                        """
                    )
                }
            }
        }
        stage('Dynamic') {
            when {
                branch 'feature/multi/*'
            }
            steps {
                echo ("${env.STAGE_NAME}")
            }
        }
        stage('Test') {
            when {
                equals expected: true,
                actual: params.RUN_TEST
            }
            steps{
                dir('pipeline'){
                    script {
                        def array = ["printer", "scanner", "main"]
                        for (element in array) {
                            env.TEST_OUTPUT_ENV += bat ( script: "Tests.bat ${element}", returnStdout: true).trim()
                            env.TEST_OUTPUT_ENV += "\n"
                        }
                    }
                }
            }
        }
        stage('Publish') {
            steps {
                script {
                    zip archive: true, defaultExcludes: false, dir: 'pipeline', exclude: '', glob: '', zipFile: "${params.ARTIFACT_NAME}.zip"
                }
                rtUpload (
                    serverId: 'jfrog_artifactory',
                    spec: """{
                          "files": [
                           {
                               "pattern": "${params.ARTIFACT_NAME}.zip",
                               "target": "generic-local/release/danilo/${env.BUILD_ID}/"
                           }
                          ]
                    }"""
                )
            }
        }
    }
    post {
        failure {
            script{
                if (params.SEND_EMAIL == true){
                    notify(env.JOB_NAME, env.BUILD_ID, currentBuild.currentResult, env.TEST_OUTPUT_ENV)
                }
            }
        }
        unstable {
            script{
                if (params.SEND_EMAIL == true){
                    notify(env.JOB_NAME, env.BUILD_ID, currentBuild.currentResult, env.TEST_OUTPUT_ENV)
                }
            }
        }
        success {
            script{
                if (params.SEND_EMAIL == true){
                    notify(env.JOB_NAME, env.BUILD_ID, currentBuild.currentResult, env.TEST_OUTPUT_ENV)
                }
            }
        }
    }
}


