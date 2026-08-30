// jbfinance CI pipeline
// -----------------------------------------------------------------------------
// Runs unchanged on BOTH a standalone Jenkins controller and a CloudBees CI
// managed controller. Each controller is configured (Manage Jenkins > System >
// Global properties > Environment variables) with:
//
//     CI_PROVIDER = jenkins      <- on the OSS Jenkins controller
//     CI_PROVIDER = cloudbees    <- on the CloudBees CI managed controller
//
// so each one reports a DISTINCT GitHub check ("jenkins/ci" vs "cloudbees/ci").
// Wire the repo to each controller as a Multibranch Pipeline using the
// "GitHub Branch Source" plugin + a GitHub App credential. Branch protection on
// `main` then requires both checks to pass plus one approving review before the
// PR can be merged.
// -----------------------------------------------------------------------------

def provider = (env.CI_PROVIDER ?: 'jenkins').toLowerCase()
def checkName = provider == 'cloudbees' ? 'cloudbees/ci' : 'jenkins/ci'

pipeline {
  agent {
    docker {
      image 'node:20-bookworm'
      // Playwright needs a writable home and a larger /dev/shm for Chromium.
      args '-u root:root --shm-size=1g'
    }
  }

  options {
    timeout(time: 30, unit: 'MINUTES')
    disableConcurrentBuilds(abortPrevious: true)
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '5'))
    timestamps()
  }

  environment {
    CI = 'true'
    HOME = "${env.WORKSPACE}"
    NPM_CONFIG_CACHE = "${env.WORKSPACE}/.npm-cache"
    // Nx uses this to key remote cache / self-healing runs, if you connect Nx Cloud later.
    NX_BRANCH = "${env.CHANGE_ID ?: env.BRANCH_NAME}"
  }

  stages {
    stage('Report: in progress') {
      steps {
        script { reportCheck(checkName, 'IN_PROGRESS', null, 'Build started') }
      }
    }

    stage('Install') {
      steps {
        sh 'node --version && npm --version'
        sh 'npm ci --legacy-peer-deps'
        sh 'npx playwright install --with-deps chromium'
      }
    }

    stage('Resolve Nx range') {
      steps {
        script {
          if (env.CHANGE_TARGET) {
            // Pull request build: only touch projects affected vs the target branch.
            sh "git fetch --no-tags --depth=200 origin ${env.CHANGE_TARGET}"
            env.NX_RANGE = "affected --base=origin/${env.CHANGE_TARGET} --head=HEAD"
          } else {
            // Branch / main build: run everything.
            env.NX_RANGE = 'run-many'
          }
          echo "nx ${env.NX_RANGE} -t <target>"
        }
      }
    }

    stage('Lint · Test · Typecheck · Build') {
      steps {
        sh "npx nx ${env.NX_RANGE} -t lint test typecheck build --parallel=3"
      }
    }

    stage('E2E') {
      steps {
        sh "npx nx ${env.NX_RANGE} -t e2e --parallel=1"
      }
    }
  }

  post {
    success  { script { reportCheck(checkName, null, 'SUCCESS', 'All checks passed') } }
    failure  { script { reportCheck(checkName, null, 'FAILURE', 'Build failed') } }
    unstable { script { reportCheck(checkName, null, 'FAILURE', 'Build unstable') } }
    always {
      junit testResults: '**/test-output/**/*.xml, **/junit*.xml, **/*junit.xml', allowEmptyResults: true
      archiveArtifacts artifacts: 'dist/**, **/playwright-report/**, **/test-output/**', allowEmptyArchive: true
    }
  }
}

// Publish a rich GitHub check via the "GitHub Checks API" plugin when a GitHub
// App credential is configured on the Multibranch source. If the plugin or
// credential is not present yet, fall back silently to the commit status the
// GitHub Branch Source plugin posts on its own.
def reportCheck(String name, String status, String conclusion, String summary) {
  try {
    if (status) {
      publishChecks name: name, status: status, title: name, summary: summary,
        detailsURL: env.BUILD_URL
    } else {
      publishChecks name: name, status: 'COMPLETED', conclusion: conclusion,
        title: name, summary: summary, detailsURL: env.BUILD_URL
    }
  } catch (ignored) {
    echo "publishChecks unavailable - relying on GitHub Branch Source commit status (${ignored.message})"
  }
}
