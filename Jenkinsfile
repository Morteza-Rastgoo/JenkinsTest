/**
 * Jenkins CI/CD pipe line
 * This script is written in groovy and it contains all the necessary steps to build, lint check, test
 * and release the application on Jenkins server.
 * For simplicity we only use one pipeline with different conditions inside the code, for example to not
 * publish the app when build is from a pull request, etc.
 * To enable syntaxt highlighting Android studio go to Preferences/Settings > Edit > File Types ->
 * Groovy -> and add Jenkinsfile* to the list.
 */


library identifier: 'jenkins-shared@master', retriever: modernSCM(
        [$class: 'GitSCMSource',
         remote: 'https://github.com/MobodidTech/jenkins-shared.git',
        ])

pipeline {
    environment {
        appName = "JenkinsTest"
        versionName = ''
        commitId = ''
    }

    agent {
        docker {
            image 'android-agent'
            //todo share args
//            args '-v /Users/imac/android-sdk:/opt/android-sdk'
//            args '-v /android-sdk:/opt/android-sdk -v /android-cache:/root/.android -v /gradle-cache:/root/.gradle'
        }
    }

    stages {
        stage("Init"){
            steps {
                script {
                    //./gradlew -q printVersion should not be the first gradle command to run sine the first command
                    //prints welcome to gradle as well!
                    sh './gradlew -q printVersion'

                    commitId = sh(returnStdout: true, script: 'git rev-parse HEAD')
                    log("commitId: $commitId")

                    def versionCode = sh(
                            script: './gradlew -q printVersion',
                            returnStdout: true
                    ).trim()
                    versionName = "v${versionCode}"

                    if(gitlab.isMergeRequest()){
                        currentBuild.displayName = "PR-${gitlab.getMergeRequestId()}"

                    } else {
                        currentBuild.displayName = "${BUILD_NUMBER}_${versionName}"

                    }
                }
            }
        }

//        stage("Lint check"){
//            steps {
//                script {
//                    sh './gradlew lintRelease --stacktrace --no-daemon'
//                }
//            }
//        }

        //todo tests are commented to speed up build while we don't have any tests. this should be uncommented.
//        stage("Junit tests"){
//            steps {
//                sh './gradlew test'
//            }
//        }

        stage('build') {
            steps {
                //todo make choosing right version of build-tools dynamic

                //building the apk file.
                sh './gradlew assembleRelease --stacktrace --no-daemon'
                sh '$ANDROID_HOME/build-tools/28.0.3/zipalign -f -v 4 app/build/outputs/apk/release/app-release-unsigned.apk app/build/outputs/apk/release/app-release-unsigned-aligned.apk'

                //build the test apk required for running instrumental tests.
                sh './gradlew assembleAndroidTest -DtestBuildType=release --stacktrace --no-daemon'
                sh '$ANDROID_HOME/build-tools/28.0.3/zipalign -f -v 4 app/build/outputs/apk/androidTest/release/app-release-androidTest.apk app/build/outputs/apk/androidTest/release/app-release-androidTest-aligned.apk'

                //sign both apk files
                withCredentials([file(credentialsId: 'unitedbitAndroidKeyStore', variable: 'KS_FILE'),
                                 string(credentialsId: 'unitedbitAndroidKeyStorePassword', variable: 'PASSWORD')]) {

                    sh '$ANDROID_HOME/build-tools/28.0.3/apksigner sign --ks $KS_FILE --ks-pass pass:$PASSWORD --key-pass pass:$PASSWORD --ks-key-alias unitedbit app/build/outputs/apk/release/app-release-unsigned-aligned.apk'
                    sh '$ANDROID_HOME/build-tools/28.0.3/apksigner sign --ks $KS_FILE --ks-pass pass:$PASSWORD --key-pass pass:$PASSWORD --ks-key-alias unitedbit app/build/outputs/apk/androidTest/release/app-release-androidTest-aligned.apk'

                    sh 'mv app/build/outputs/apk/release/app-release-unsigned-aligned.apk app/build/outputs/apk/release/'+ getReleaseName()

                    script {
                        echo "VersionInfo: ${versionName}"
                    }
                }
            }
        }

//        stage("Instrumental Tests"){
//            steps {
//                withCredentials([file(credentialsId: 'firebaseJsonFile', variable: 'FIREBASE_JSON_FILE'),
//                                 string(credentialsId: 'firebaseEmail', variable: 'FIREBASE_EMAIL')]) {
//
//                    //todo project id as credentials
//                    sh 'gcloud auth activate-service-account --key-file=$FIREBASE_JSON_FILE'
//                }
//
//                sh "gcloud firebase test android run --app app/build/outputs/apk/release/${getReleaseName()} --test app/build/outputs/apk/androidTest/release/app-release-androidTest-aligned.apk --project=jenkinstest-5a4e5"
//            }
//        }

        stage("Upload"){
            steps {
                script {
                    if(gitlab.isMergeRequest()){
                        sendTelegram("PR succeed for ${appName} Android\n${gitlab.getMergeRequestInfo()}")
                    } else {
                        upload()
                    }
                }
            }
        }
    }

    post {
        failure {
            script {
                if (gitlab.isMergeRequest()) {
                    sendTelegram("PR failed for ${appName} Android\n ${gitlab.getMergeRequestInfo()}")

                } else {
                    sendTelegram("Build failed for ${appName} Android ${currentBuild.displayName}\n" +
                            "Checkout Jenkins console for more information.")
                }
            }
        }
    }
}

def getReleaseName(){
    "${appName}_${versionName}.apk"
}

//For now it only updates to Telegram.
def upload(){
    sh "cp app/build/outputs/apk/release/${getReleaseName()} /var/jenkins_artifacts"
    def fileLocation = "${getServerIP()}/${getReleaseName()}"

    sendTelegram("Build successful for ${appName} Android ${currentBuild.displayName}\n" +
            "$fileLocation \nPlease do not send this APK to anyone outside Mobodid without consulting dev team.")

    sh 'echo $fileLocation'
}

def getServerIP(){
    def splittedUrl = "${env.BUILD_URL}".split('/')
    splittedUrl[0] + '//' + splittedUrl[2].split(':')[0]
}

def sendTelegram(message){
    telegram.sendTelegram(message)
}

def log(message){
    sh "echo $message"
}


