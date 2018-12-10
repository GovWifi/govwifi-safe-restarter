class Globals {
  static boolean userInput = true
  static boolean didTimeout = false
}

pipeline {
  agent any
  stages {
    stage('Test') {
      steps {
        sh 'make test'
      }
    }

    stage('Publish stable tag') {
      when{
        branch 'master'
      }

      steps {
        publishStableTag()
      }
    }

    stage('Deploy to staging') {
      when{
        branch 'master'
      }

      steps {
        deploy('staging', false, 1)
      }
    }

    stage('Deploy to production') {
      when{
        branch 'master'
      }

      steps {
        deploy('production', true, 2)
      }
    }
  }

  post {
    always {
      sh 'make stop'
    }
  }
}


def deploy(deploy_environment, requires_confirmation, desired_count) {
  if(deployCancelled()) {
    return
  }

  echo "${deploy_environment}"
  try {
    timeout(time: 5, unit: 'MINUTES') {
      if(requires_confirmation) {
        input "Do you want to deploy to ${deploy_environment}?"
      }

      // Jenkins does a fetch without tags during setup this means
      // we need to run git fetch again here before we can checkout stable
      sh('git fetch')
      sh('git checkout stable')

      docker.withRegistry(env.AWS_ECS_API_REGISTRY) {
        sh("eval \$(aws ecr get-login --no-include-email)")
        def appImage = docker.build(
          "govwifi/safe-restarter:${deploy_environment}",
          "--build-arg BUNDLE_INSTALL_CMD='bundle install --without test' ."
        )
        appImage.push()
      }
    }
  } catch(err) { // timeout reached or input false
    def user = err.getCauses()[0].getUser()

    if('SYSTEM' == user.toString()) { // SYSTEM means timeout.
      Globals.didTimeout = true
      echo "Release window timed out, to deploy please re-run"
    } else {
      Globals.userInput = false
      echo "Aborted by: [${user}]"
    }
  }
}

def publishStableTag() {
  sshagent(credentials: ['govwifi-jenkins']) {
    sh('export GIT_SSH_COMMAND="ssh -oStrictHostKeyChecking=no"')
    sh("git tag -f stable HEAD")
    sh("git push git@github.com:alphagov/govwifi-safe-restarter.git --force --tags")
  }
}

def deployCancelled() {
  if(Globals.didTimeout || Globals.userInput == false) {
    return true
  }
}
