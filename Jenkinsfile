pipeline {
  agent { label 'ansible && rocky9' }
  options {
    skipDefaultCheckout(true)
    timestamps()
  }
  triggers { pollSCM('H/2 * * * *') }
  environment {
    BUILD_PAYMENTS = 'false'
    BUILD_CATALOG = 'false'
  }
  stages {
    stage('Checkout and detect changed apps') {
      steps {
        checkout scm
        script {
          echo "Source commit: ${env.GIT_COMMIT}"
          def changedFiles
          if (env.GIT_PREVIOUS_SUCCESSFUL_COMMIT?.trim()) {
            changedFiles = sh(script: "git diff --name-only ${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT} ${env.GIT_COMMIT}", returnStdout: true).trim().tokenize('\n')
          } else {
            changedFiles = sh(script: 'git ls-files apps', returnStdout: true).trim().tokenize('\n')
          }
          env.BUILD_PAYMENTS = changedFiles.any { it.startsWith('apps/payments/') } ? 'true' : 'false'
          env.BUILD_CATALOG = changedFiles.any { it.startsWith('apps/catalog/') } ? 'true' : 'false'
          echo "payments 변경: ${env.BUILD_PAYMENTS}, catalog 변경: ${env.BUILD_CATALOG}"
        }
      }
    }
    stage('Build and push changed apps') {
      when {
        expression { env.BUILD_PAYMENTS == 'true' || env.BUILD_CATALOG == 'true' }
      }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-write',
          usernameVariable: 'DH_USER',
          passwordVariable: 'DH_TOKEN'
        )]) {
          sh '''
            set -eu
            set +x
            printf '%s' "$DH_TOKEN" | podman login docker.io \
              --username "$DH_USER" --password-stdin
            for APP in payments catalog; do
              SHOULD_BUILD="$BUILD_PAYMENTS"
              if [ "$APP" = catalog ]; then SHOULD_BUILD="$BUILD_CATALOG"; fi
              if [ "$SHOULD_BUILD" != true ]; then continue; fi
              APP_DIR="apps/$APP"
              IMAGE_REPO="docker.io/$DH_USER/$APP"
              IMAGE_TAG="build-$BUILD_NUMBER"
              test -f "$APP_DIR/Dockerfile"
              test -f "$APP_DIR/index.html"
              grep -qi "$APP 버전" "$APP_DIR/index.html"
              podman build --pull -f "$APP_DIR/Dockerfile" \
                --tag "$IMAGE_REPO:$IMAGE_TAG" \
                --tag "$IMAGE_REPO:dev" "$APP_DIR"
              podman push "$IMAGE_REPO:$IMAGE_TAG"
              podman push "$IMAGE_REPO:dev"
              echo "$APP 업로드 완료: $IMAGE_TAG 및 dev"
            done
            podman logout docker.io
          '''
        }
      }
    }
  }
  post {
    always { sh 'podman logout docker.io >/dev/null 2>&1 || true' }
  }
}