pipeline {
  agent { label 'ansible && rocky9' }

  options {
    skipDefaultCheckout(true)
    timestamps()
  }

  triggers {
    pollSCM('H/2 * * * *')
  }

  environment {
    BUILD_PAYMENTS = 'false'
    BUILD_CATALOG = 'false'
  }

  stages {

    stage('Checkout and detect changed apps') {
      steps {
        checkout scm

        script {

          echo "Current Commit : ${env.GIT_COMMIT}"
          echo "Previous Commit: ${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT}"

          if (env.GIT_PREVIOUS_SUCCESSFUL_COMMIT?.trim()) {

            def changedFiles = sh(
              script: """
                git diff --name-only \
                ${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT} \
                HEAD
              """,
              returnStdout: true
            ).trim().tokenize('\n')

            echo "Changed files: ${changedFiles}"

            env.BUILD_PAYMENTS =
              changedFiles.any { it.startsWith('apps/payments/') } ?
              'true' : 'false'

            env.BUILD_CATALOG =
              changedFiles.any { it.startsWith('apps/catalog/') } ?
              'true' : 'false'

          } else {

            echo "First build detected."
            echo "Building all applications."

            env.BUILD_PAYMENTS = 'true'
            env.BUILD_CATALOG = 'true'
          }

          echo "payments 변경: ${env.BUILD_PAYMENTS}"
          echo "catalog 변경: ${env.BUILD_CATALOG}"
        }
      }
    }

    stage('Build and push changed apps') {
      when {
        expression {
          env.BUILD_PAYMENTS == 'true' ||
          env.BUILD_CATALOG == 'true'
        }
      }

      steps {

        withCredentials([
          usernamePassword(
            credentialsId: 'dockerhub-write',
            usernameVariable: 'DH_USER',
            passwordVariable: 'DH_TOKEN'
          )
        ]) {

          sh '''
            set -eu
            set +x

            printf '%s' "$DH_TOKEN" | \
              podman login docker.io \
              --username "$DH_USER" \
              --password-stdin

            for APP in payments catalog
            do

              SHOULD_BUILD="$BUILD_PAYMENTS"

              if [ "$APP" = "catalog" ]; then
                SHOULD_BUILD="$BUILD_CATALOG"
              fi

              if [ "$SHOULD_BUILD" != "true" ]; then
                echo "Skip $APP"
                continue
              fi

              APP_DIR="apps/$APP"

              IMAGE_REPO="docker.io/$DH_USER/$APP"

              IMAGE_TAG="build-$BUILD_NUMBER"

              echo "Building $APP"

              test -f "$APP_DIR/Dockerfile"

              podman build \
                --pull \
                -f "$APP_DIR/Dockerfile" \
                -t "$IMAGE_REPO:$IMAGE_TAG" \
                -t "$IMAGE_REPO:dev" \
                "$APP_DIR"

              podman push "$IMAGE_REPO:$IMAGE_TAG"
              podman push "$IMAGE_REPO:dev"

              echo "$APP 업로드 완료"
            done

            podman logout docker.io
          '''
        }
      }
    }
  }

  post {
    always {
      sh 'podman logout docker.io >/dev/null 2>&1 || true'
    }
  }
}