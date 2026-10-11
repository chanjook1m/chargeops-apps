pipeline {
  agent { label 'ansible && rocky9' }

  options {
    skipDefaultCheckout(true)
    timestamps()
  }

  triggers {
    pollSCM('H/2 * * * *')
  }

  stages {
    stage('Checkout and detect changed apps') {
      steps {
        script {
          def checkoutInfo = checkout scm

          // Jenkins가 실제로 체크아웃한 커밋을 워크스페이스에서 읽는다.
          def currentCommit = sh(
            script: 'git rev-parse HEAD',
            returnStdout: true
          ).trim()

          def previousCommit =
            checkoutInfo?.get('GIT_PREVIOUS_SUCCESSFUL_COMMIT')?.toString()?.trim()

          if (!previousCommit) {
            previousCommit =
              env.GIT_PREVIOUS_SUCCESSFUL_COMMIT?.toString()?.trim()
          }

          if (previousCommit &&
              !(previousCommit ==~ /[0-9a-fA-F]{40,64}/)) {
            echo 'Previous successful commit 값이 SHA 형식이 아니므로 전체 빌드합니다.'
            previousCommit = ''
          }

          env.CI_CURRENT_COMMIT = currentCommit
          env.CI_PREVIOUS_SUCCESSFUL_COMMIT = previousCommit ?: ''

          // 동적으로 설정하므로 Jenkinsfile의 environment 블록에 선언하지 않는다.
          env.BUILD_PAYMENTS = 'false'
          env.BUILD_CATALOG = 'false'

          echo "Branch: ${env.BRANCH_NAME ?: '(multibranch 아님)'}"
          echo "Current Commit: ${env.CI_CURRENT_COMMIT}"

          if (previousCommit) {
            echo "Previous Successful Commit: ${previousCommit}"

            // 이전 커밋이 현재 커밋의 조상인지 확인한다.
            def isAncestor = sh(
              script: "git merge-base --is-ancestor '${previousCommit}' '${currentCommit}'",
              returnStatus: true
            )

            if (isAncestor == 0) {
              def diffOutput = sh(
                script: "git diff --name-only '${previousCommit}' '${currentCommit}'",
                returnStdout: true
              ).trim()

              def changedFiles = diffOutput ? diffOutput.readLines() : []
              echo "Changed files: ${changedFiles}"

              env.BUILD_PAYMENTS =
                changedFiles.any {
                  it == 'apps/payments' || it.startsWith('apps/payments/')
                } ? 'true' : 'false'

              env.BUILD_CATALOG =
                changedFiles.any {
                  it == 'apps/catalog' || it.startsWith('apps/catalog/')
                } ? 'true' : 'false'
            } else {
              echo '이전 커밋을 현재 브랜치 이력에서 찾지 못해 전체 빌드합니다.'
              env.BUILD_PAYMENTS = 'true'
              env.BUILD_CATALOG = 'true'
            }
          } else {
            echo '이 브랜치의 이전 성공 빌드가 없어 전체 빌드합니다.'
            env.BUILD_PAYMENTS = 'true'
            env.BUILD_CATALOG = 'true'
          }

          echo "BUILD_PAYMENTS=${env.BUILD_PAYMENTS}"
          echo "BUILD_CATALOG=${env.BUILD_CATALOG}"
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
            credentialsId: 'cicd',
            usernameVariable: 'DH_USER',
            passwordVariable: 'DH_TOKEN'
          )
        ]) {
          sh '''
            set -eu
            set +x

            printf '%s' "$DH_TOKEN" |
              podman login docker.io \
                --username "$DH_USER" \
                --password-stdin

            IMAGE_TAG="git-${CI_CURRENT_COMMIT:0:12}"

            for APP in payments catalog
            do
              if [ "$APP" = "payments" ]; then
                SHOULD_BUILD="$BUILD_PAYMENTS"
              else
                SHOULD_BUILD="$BUILD_CATALOG"
              fi

              if [ "$SHOULD_BUILD" != "true" ]; then
                echo "Skip $APP"
                continue
              fi

              APP_DIR="apps/$APP"
              IMAGE_REPO="docker.io/$DH_USER/$APP"

              echo "Building $APP with tag $IMAGE_TAG"
              test -f "$APP_DIR/Dockerfile"

              podman build \
                --pull \
                -f "$APP_DIR/Dockerfile" \
                -t "$IMAGE_REPO:$IMAGE_TAG" \
                "$APP_DIR"

              podman push "$IMAGE_REPO:$IMAGE_TAG"

              # dev 태그는 main 브랜치 빌드에만 갱신한다.
              if [ "${BRANCH_NAME:-}" = "main" ]; then
                podman tag "$IMAGE_REPO:$IMAGE_TAG" "$IMAGE_REPO:dev"
                podman push "$IMAGE_REPO:dev"
              fi

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