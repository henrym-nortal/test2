pipeline {
  agent any

  options {
    ansiColor("xterm")
    buildDiscarder(logRotator(numToKeepStr: "20"))
    disableConcurrentBuilds()
    skipStagesAfterUnstable()
    timestamps()
  }

  environment {
    APP_NAME = 'ria-storybook'
    DOCKER_IMAGE_TAG = getStorybookImageTag()
    CYPRESS_DOWNLOAD_MIRROR = "https://nexus.riaint.ee/repository/cypress-raw-proxy/"
    PUBLIC_REGISTRY = "nexus.riaint.ee:8500"
    HARBOR_REGISTRY = "harbor.riaint.ee"
    DOCKER_IMAGE = "riaee/sun-ria-storybook"
    HARBOR_DOCKER_IMAGE = "${HARBOR_REGISTRY}/sun/sun-${APP_NAME}"
    HUSKY = 0
    NAMESPACE = "sun"
  }

  parameters {
    booleanParam(name: "PUBLISH_STYLES", description: "Publish styles (tag exists, but wasn't published before)", defaultValue: false)
    booleanParam(name: "PUBLISH_UI", description: "Publish ui (tag exists, but wasn't published before)", defaultValue: false)
    booleanParam(name: "PUBLISH_ICONS", description: "Publish icons (tag exists, but wasn't published before)", defaultValue: false)
  }

  stages {
    stage("Skip?") {
      steps {
        script {
          result = sh (script: "git log -1 | grep '.*\\[skip ci\\].*'", returnStatus: true)
          if (result == 0) {
            echo "'skip ci' spotted in git commit. Aborting."
            env.skip_ci = true
          }
        }
      }
    }
    stage('CI') {
      agent {
        docker {
          image 'nexus.riaint.ee:8500/node:lts'
        }
      }
      environment {
        HOME = "${env.WORKSPACE}"
      }
      stages {
        stage('deploy storybook') {
          environment {
            GITHUB = credentials('jenkins-cvi-github')
          }
          steps {
            script {
              sh '''
              git config --global user.name 'sun-release-bot'
              git config --global user.email 'sun-release-bot@example.com'
              git remote set-url origin https://${GITHUB_USR}:${GITHUB_PSW}@github.com/henrymae/test.git
              '''

              sh '''
              npm config set registry https://nexus.riaint.ee/repository/npm-public/
              npm install
              ls
              gh-pages -d docs/build/html --message 'chore: update github pages [skip ci]'
              '''

              sh '''
              #git add docs
              #git commit -m \"chore: update github pages [skip ci] \"
              #git push origin release --tags
              '''
            }
          }
        }
      }
    }
  }
  post {
    always {
      cleanWs()
    }
  }
}

def getStorybookImageTag() {
  if (isMaster()) {
    return env.storybook_library_version == null ? (getVersion("storybook") + '.1') : env.storybook_library_version
  }
  return env.CHANGE_BRANCH == null ? env.BRANCH_NAME : env.CHANGE_BRANCH
}

def getVersion(String libraryName) {
  def props = readJSON file: "libs/${libraryName}/package.json"
  return props.version
}

def isMaster() {
  return env.BRANCH_NAME == "master"
}

def affected(String project) {
  return env.affected_libraries.split(', ').contains(project)
}

void deploy() {
  checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false,
  extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'deployment']], submoduleCfg: [],
  userRemoteConfigs: [[credentialsId: 'jenkins-bitbucket-webhook', url: 'https://stash.ria.ee/scm/sun/helm-charts.git']]])

  def ns = "sun-${ENV}"

  docker.image('nexus.riaint.ee:8500/alpine/helm:3.9.4').inside('-u 0 --entrypoint= ') {
    dir("deployment/${APP_NAME}") {
      sh script: """
      helm version
      helm list -n ${NAMESPACE}
      helm lint . -f values.yaml -f values-${ENV}.yaml

      helm upgrade -n ${NAMESPACE} ${APP_NAME} . -f values.yaml  -f values-${ENV}.yaml --set image.tag=${DOCKER_IMAGE_TAG} --install --create-namespace --atomic --timeout 5m0s
      helm list -n ${NAMESPACE} | grep ${APP_NAME}
      """, label: "Deploy application";
    }
  }
}

def waitForUserApproval(Integer secondsToWait, String message) {
  def approval = false

  try {
    timeout(time: secondsToWait, unit: 'SECONDS') {
      approval = input(
        id: 'Proceed1',
        message: message,
        parameters: [[$class: 'BooleanParameterDefinition', defaultValue: false, description: '', name: 'APPROVAL']]
      )
    }
  } catch(org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
    echo "Approval timout or aborted by user"
  }
  return approval;
}
