def dockerBuildTag = 'latest'
def mobileDepositUiImage = null
def buildVersion = null
def short_commit = null

env.DOCKER_HUB_USER = 'beedemo'
env.DOCKER_CREDENTIAL_ID = 'docker-hub-beedemo'

if(env.BRANCH_NAME=="master"){
    properties([pipelineTriggers(triggers: [[$class: 'DockerHubTrigger', options: [[$class: 'TriggerOnSpecifiedImageNames', repoNames: ['beedemo/mobile-deposit-api'] as Set]]]]),
                [$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '5']]])
}


node('docker-compose') {
    checkout scm
    stage('build') {
        sh('git rev-parse HEAD > GIT_COMMIT')
        git_commit=readFile('GIT_COMMIT')
        short_commit=git_commit.take(7)
        sh 'docker run -i --rm -v "$PWD":/usr/src/mobile-deposit-ui -w /usr/src/mobile-deposit-ui maven:3.3-jdk-8 mvn -Dmaven.repo.local=/data/mvn/repo clean package -DskipTests'

        //get new version of application from pom
        def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
        if (matcher) {
            buildVersion = matcher[0][1]
            echo "Released version ${buildVersion}"
        }
        matcher = null
    }
    stage('functional-test') {
        try {
            sh 'docker-compose up -d'
            parallel(
                "firefox": {
                    sh 'docker pull selenoid/firefox:46.0'
                    sh 'docker run -i --rm -p 8081:8081 -v "$PWD":/usr/src/mobile-deposit-ui -w /usr/src/mobile-deposit-ui maven:3.3-jdk-8 mvn -Dmaven.repo.local=/data/mvn/repo verify -DargLine="-Dtest.host=172.17.0.1 -Dtest.browser.name=firefox -Dtest.browser.version=46.0 -Dserver.port=8081"'
                },
                "chrome": {
                    sh 'docker pull selenoid/chrome:59.0'
                    sh 'docker run -i --rm -p 8082:8082 -v "$PWD":/usr/src/mobile-deposit-ui -w /usr/src/mobile-deposit-ui maven:3.3-jdk-8 mvn -Dmaven.repo.local=/data/mvn/repo verify -DargLine="-Dtest.host=172.17.0.1 -Dtest.browser.name=chrome -Dtest.browser.version=59.0 -Dserver.port=8082"'
                }, failFast: true
            )
        } catch(x) {
            //error
            throw x
        } finally {
            sh 'docker-compose down'
        }
    }
    //docker tag to be used for build, push and run
    def dockerTag = "${env.BUILD_NUMBER}-${short_commit}"
    //build image and deploy to staging
    stage('build docker image') {
        dir('target') {
            mobileDepositUiImage = docker.build "beedemo/mobile-deposit-ui-stage:${dockerTag}"
        }
    }

    stage('deploy to staging') {
        //use withDockerRegistry to make sure we are logged in to docker hub registry
        withDockerRegistry(registry: [credentialsId: 'docker-hub-beedemo']) {
          mobileDepositUiImage.push()
        }
        dockerDeploy("docker-cloud","${DOCKER_HUB_USER}", 'mobile-deposit-ui-stage', 82, 8080, "$dockerTag")
    }
}


stage('awaiting approval') {
    //put input step outside of node so it doesn't tie up a slave
    timeout(time: 10, unit: 'MINUTES') {
        input 'UI Staged at http://bank.beedemo.net:82/deposit - Proceed with Production Deployment?'
    }
}

if(env.BRANCH_NAME=="master") {//only deploy master branch to prod
    stage('deploy to production') {
        node('docker-cloud') {
            def dockerTag = "${env.BUILD_NUMBER}-${short_commit}"
            sh "docker tag beedemo/mobile-deposit-ui-stage:${dockerTag} beedemo/mobile-deposit-ui:${dockerTag}"
            //use withDockerRegistry to make sure we are logged in to docker hub registry
            withDockerRegistry(registry: [credentialsId: 'docker-hub-beedemo']) {
              sh "docker push beedemo/mobile-deposit-ui:${dockerTag}"
            }
            dockerDeploy("docker-cloud","${DOCKER_HUB_USER}", 'mobile-deposit-ui', 80, 8080, "$dockerTag")
        }
    }
}
