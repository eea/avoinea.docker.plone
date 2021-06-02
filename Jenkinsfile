pipeline {
  agent any

  environment {
    GIT_NAME = "eea.docker.plonesaas"
  }

  parameters {
    string(defaultValue: '', description: 'Run tests with GIT_BRANCH env enabled', name: 'TARGET_BRANCH')
  }
  
  stages {
    stage('Build & Test') {      
      steps {
        node(label: 'docker-host') {
          script {
            try {
              checkout scm
              sh '''docker build -t ${BUILD_TAG,,} .'''
              sh '''docker run -d --name=${BUILD_TAG,,} ${BUILD_TAG,,} fg'''
              sh '''docker run -i --rm --link=${BUILD_TAG,,}:plone --name=${BUILD_TAG,,}-test --entrypoint /plone/instance/bin/zopepy ${BUILD_TAG,,} -c "from six.moves.urllib.request import urlopen; import time; time.sleep(45); con = urlopen('http://plone:8080'); print(con.read())"'''
              sh '''./test/run.sh ${BUILD_TAG,,}'''
            } finally {                
                sh script: "docker logs ${BUILD_TAG,,}", returnStatus: true
                sh script: "docker stop ${BUILD_TAG,,}", returnStatus: true
                sh script: "docker rm -v ${BUILD_TAG,,}", returnStatus: true
                sh script: "docker rmi ${BUILD_TAG,,}", returnStatus: true
            }
          }
        }
      }
    }

    stage('Release on tag creation') {
      when {
        buildingTag()
      }
      steps{
        node(label: 'docker') {
          withCredentials([string(credentialsId: 'eea-jenkins-token', variable: 'GITHUB_TOKEN'), string(credentialsId: 'plonesaas-devel-trigger', variable: 'TRIGGER_URL'), string(credentialsId: 'plonesaas-trigger', variable: 'TRIGGER_MAIN_URL'),usernamePassword(credentialsId: 'jekinsdockerhub', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
           sh '''docker pull eeacms/gitflow; docker run -i --rm --name="$BUILD_TAG-plonesaas" -e GIT_BRANCH="$BRANCH_NAME" -e GIT_NAME="$GIT_NAME" -e GIT_TOKEN="$GITHUB_TOKEN" -e TRIGGER_MAIN_URL="$TRIGGER_MAIN_URL" -e DOCKERHUB_USER="$DOCKERHUB_USER" -e DOCKERHUB_PASS="$DOCKERHUB_PASS" -e DOCKERHUB_REPO="eeacms/plonesaas" -e DEPENDENT_DOCKERFILE_URL="devel/Dockerfile"  -e TRIGGER_RELEASE="eeacms/plonesaas-devel;$TRIGGER_URL" -e GITFLOW_BEHAVIOR="RUN_ON_TAG" eeacms/gitflow'''
         }
        }
      }
    }

  }

  post {
    always {
      cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenSuccess: true, cleanWhenUnstable: true, deleteDirs: true)
    }
    changed {
      script {
        def details = """<h1>${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}</h1>
                         <p>Check console output at <a href="${env.BUILD_URL}/display/redirect">${env.JOB_BASE_NAME} - #${env.BUILD_NUMBER}</a></p>
                      """
        emailext(
        subject: '$DEFAULT_SUBJECT',
        body: details,
        attachLog: true,
        compressLog: true,
        recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'CulpritsRecipientProvider']]
        )
      }
    }
  }
}
