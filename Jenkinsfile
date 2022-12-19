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
          image 'node:latest'
        }
      }
      environment {
        HOME = "${env.WORKSPACE}"
      }
      stages {
        stage('publish storybook') {
          environment {
            KOODIVARAMU_TOKEN = credentials('jenkins-cvi-koodivaramu')
          }
          steps {
            script {
              sh 'echo "registry=https://koodivaramu.eesti.ee/api/v4/projects/433/packages/npm/" > .npmrc'
              sh "echo '_authToken=${KOODIVARAMU_TOKEN}' >> .npmrc"

              ["styles", "ui", "icons"].each {
                //if (env."previous_${it}_library_version" == env."${it}_library_version" && !params."PUBLISH_${it.toUpperCase()}") {
                //  echo "${it} version ${getVersion(it)} is already published"
                //  return
                //}
                sh "npx nx build ${it}"
                sh "npm run publish:${it}"
                echo "Published library ${it}"
              }
            }
          }
        }

        stage('deploy storybook') {
          environment {
            GITHUB_TOKEN = credentials('jenkins-cvi-github')
          }
          steps {
            script {
              sh '''
              git config --global user.name 'sun-release-bot'
              git config --global user.email 'sun-release-bot@example.com'
              git remote set-url origin https://${GITHUB_TOKEN}@github.com/henrymae/test.git
              '''

              sh '''
              npm config set registry https://nexus.riaint.ee/repository/npm-public/
              npm install
              npm run build-storybook
              npx gh-pages -d dist/storybook/storybook --message 'chore: update github pages [skip ci]'
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
