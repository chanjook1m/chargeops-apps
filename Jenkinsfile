pipeline {
  agent { label 'ansible && rocky9' }

  options {
    timestamps()
    disableConcurrentBuilds()
    skipDefaultCheckout(true)
  }

  stages {
    stage('Checkout and show commits') {
      steps {
        script {
          def checkoutInfo = checkout scm

          def currentCommit = checkoutInfo?.get('GIT_COMMIT')?.toString()?.trim()
          if (!currentCommit) {
            currentCommit = sh(
              returnStdout: true,
              script: 'git rev-parse HEAD'
            ).trim()
          }

          def previousSuccessfulCommit =
            checkoutInfo?.get('GIT_PREVIOUS_SUCCESSFUL_COMMIT')?.toString()?.trim()

          if (!previousSuccessfulCommit) {
            previousSuccessfulCommit =
              env.GIT_PREVIOUS_SUCCESSFUL_COMMIT?.toString()?.trim()
          }

          env.CI_CURRENT_COMMIT = currentCommit
          env.CI_PREVIOUS_SUCCESSFUL_COMMIT =
            previousSuccessfulCommit ?: ''

          echo "BRANCH_NAME=${env.BRANCH_NAME ?: '(not a multibranch build)'}"
          echo "CI_CURRENT_COMMIT=${env.CI_CURRENT_COMMIT}"

          if (env.CI_PREVIOUS_SUCCESSFUL_COMMIT) {
            echo "CI_PREVIOUS_SUCCESSFUL_COMMIT=${env.CI_PREVIOUS_SUCCESSFUL_COMMIT}"
          } else {
            echo 'CI_PREVIOUS_SUCCESSFUL_COMMIT=(이 브랜치에 이전 성공 빌드가 없음)'
          }
        }

        sh '''
          set -eu
          test "$(git rev-parse HEAD)" = "$CI_CURRENT_COMMIT"
          git log -1 --format='HEAD=%H%nSUBJECT=%s'
        '''
      }
    }

    stage('Verify agent') {
      steps {
        sh '''
          set -eu
          echo "Running on: $NODE_NAME"
          hostname
          java -version
          ansible --version
        '''
      }
    }
  }
}