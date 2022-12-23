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
    DOCKER_IMAGE_TAG = getStorybookImageTag()
    RIA_INTERNAL_NEXUS_REGISTRY = "nexus.riaint.ee:8500"
    RIA_INTERNAL_HARBOR_REGISTRY = "harbor.riaint.ee"
    KOODIVARAMU_REGISTRY = "https://koodivaramu.eesti.ee:5050/egov"
    DOCKER_IMAGE = "e-gov/cvi/storybook"
    RIA_INTERNAL_DOCKER_IMAGE = "${RIA_INTERNAL_HARBOR_REGISTRY}/sun/cvi-storybook"
    HUSKY = 0
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
      when {
        expression { !env.skip_ci }
      }
      agent {
        docker {
          image 'nexus.riaint.ee:8500/node:lts'
        }
      }
      environment {
        HOME = "${env.WORKSPACE}"
      }
      stages {
        stage('npm install') {
          steps {
            sh 'npm config set registry https://nexus.riaint.ee/repository/npm-public/'
            sh "npm ci"
          }
        }
        stage('verify build') {
          steps {
            script {
              ["styles", "ui", "icons"].each {
                sh "npx nx build ${it}"
             }
            }
          }
        }
        stage('get affected projects') {
          steps {
            script {
              env.affected_libraries = sh ( script: "npx nx print-affected --base=HEAD~1 --select=projects --type=lib", returnStdout: true).trim()
              echo "Affected libraries: ${env.affected_libraries}"

              env.affected_apps = sh ( script: "npx nx print-affected --base=HEAD~1 --select=projects --type=app", returnStdout: true).trim()
              echo "Affected apps: ${env.affected_apps}"
            }
          }
        }
      }
    }
    stage('cypress tests') {
      when {
         expression { !env.skip_ci }
      }
      agent {
        docker {
          image 'nexus.riaint.ee:8500/cypress/base:18.6.0'
        }
      }
      environment {
        HOME = "${env.WORKSPACE}"
      }
      steps {
        sh 'npm config set registry https://nexus.riaint.ee/repository/npm-public/'
        sh 'npm ci'
        sh 'npm run generate-icons'
        sh 'npm run storybook:compodoc'
        sh 'npx nx run ui-e2e:e2e'
      }
      post {
        always {
          junit 'reports/cypress/*.xml'
        }
      }
    }
    stage('sonarqube analysis') {
      when {
        expression { !env.skip_ci }
      }
      agent {
        docker {
          image 'nexus.riaint.ee:8500/sonarsource/sonar-scanner-cli:4.6'
          args "-u 0 -t"
        }
      }
      steps {
        withSonarQubeEnv('RIA SonarQube') {
          sh "sonar-scanner"
        }
      }
      post {
        always {
          sh "chmod -R 777 .";
        }
      }
    }
    stage('release and publish libraries') {
      when {
        expression { !env.skip_ci }
        expression { isMain() }
      }
      agent {
        docker {
          image 'nexus.riaint.ee:8500/node:lts'
        }
      }
      environment {
        BITBUCKET = credentials('jenkins-bitbucket-webhook')
        GITHUB_TOKEN = credentials('jenkins-cvi-github')
        HOME = "${env.WORKSPACE}"
      }
      stages {
        stage('stage setup') {
          steps {
            sh '''
            npm config set registry https://nexus.riaint.ee/repository/npm-public/
            npm ci
            '''

            sh '''
            git config --global user.name 'egov-cvi-bot'
            git config --global user.email 'egov-cvi-bot@egov.ee'
            git remote set-url origin https://${GITHUB_TOKEN}@github.com/henrymae/test2.git
            #git remote set-url origin https://${GITHUB_TOKEN}@github.com/e-gov/cvi.git
            '''
          }
        }

        stage('release libraries') {
          steps {
            script {
              env.previous_styles_library_version = getVersion("styles")
              env.previous_ui_library_version = getVersion("ui")
              env.previous_icons_library_version = getVersion("icons")

              ["styles", "ui", "icons", "storybook"].each {
                try {
                  sh "npx nx run ${it}:version"
                  if (it == 'storybook') {
                    env.storybook_library_version = getVersion("storybook")
                    echo "Current storybook version: ${env.storybook_library_version}"
                  }
                } catch (e) {
                  def newVersion = getVersion(it)
                  if (env."previous_${it}_library_version" == getVersion(it)) {
                    throw e
                  }
                  def tag = "${it}-${newVersion}"
                  def tagExists = sh ( script: "git tag -l ${tag}", returnStdout: true).trim()
                  if (tagExists) {
                    sh "git tag -d ${tag}"
                    sh "git push origin :refs/tags/${tag}"
                  }
                  throw e
                }

                env.styles_library_version = getVersion("styles")
                env.ui_library_version = getVersion("ui")
                env.icons_library_version = getVersion("icons")
              }
            }
          }
        }

        stage('publish libraries') {
          environment {
            KOODIVARAMU_TOKEN = credentials('jenkins-cvi-koodivaramu')
          }
          steps {
            script {
              sh 'echo "registry=https://koodivaramu.eesti.ee/api/v4/projects/433/packages/npm/" > .npmrc'
              sh "echo '//koodivaramu.eesti.ee/api/v4/projects/433/packages/npm/:_authToken=${KOODIVARAMU_TOKEN}' >> .npmrc"

              ["styles", "ui", "icons"].each {
                sh "npx nx build ${it}"
                sh "npm run publish:${it}"
                echo "Published library ${it}"
              }
            }
          }
        }
        stage('publish storybook to github pages') {
          environment {
            GITHUB_TOKEN = credentials('jenkins-cvi-github')
          }
          steps {
            script {
              sh '''
              npm run build-storybook
              npx gh-pages -d dist/storybook/storybook --message 'chore: update github pages [skip ci]'
              '''
            }
          }
        }
      }
    }
    stage('build and publish storybook docker image') {
      when {
        expression { !env.skip_ci }
        expression { isMain() }
        //expression { affected("storybook") }
      }
      steps {
        script {
          def styles_version = env.styles_library_version ?: getVersion("styles")
          def ui_version = env.ui_library_version ?: getVersion("ui")
          def icons_version = env.icons_library_version ?: getVersion("icons")
          def dockerImage = docker.build(DOCKER_IMAGE, [
            "--build-arg node_version=${RIA_INTERNAL_NEXUS_REGISTRY}/node:lts",
            "--build-arg nginx_version=${RIA_INTERNAL_NEXUS_REGISTRY}/nginx:1.23.1-alpine",
            "--build-arg alpine_version=${RIA_INTERNAL_NEXUS_REGISTRY}/alpine:3.14",
            "--build-arg styles_version=${styles_version}",
            "--build-arg ui_version=${ui_version}",
            "--build-arg icons_version=${icons_version}",
            "-f ./libs/storybook/Dockerfile",
            "."
          ].join(" "))
          //docker.withRegistry(env.KOODIVARAMU_REGISTRY, 'koodivaramu-docker-registry') {
          //  dockerImage.push(env.DOCKER_IMAGE_TAG)
          //  dockerImage.push('latest')  123
          //}

          //docker.withRegistry("https://${KOODIVARAMU_REGISTRY}/sun", 'harbor-sun') {
          //  sh "docker tag ${DOCKER_IMAGE} ${RIA_INTERNAL_DOCKER_IMAGE}:${DOCKER_IMAGE_TAG}"
          //  sh "docker tag ${DOCKER_IMAGE} ${RIA_INTERNAL_DOCKER_IMAGE}:latest"

          //  sh "docker push ${RIA_INTERNAL_DOCKER_IMAGE}:${DOCKER_IMAGE_TAG}"
          //  sh "docker push ${RIA_INTERNAL_DOCKER_IMAGE}:latest"
          //}
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
  if (isMain()) {
    return env.storybook_library_version == null ? (getVersion("storybook") + '.1') : env.storybook_library_version
  }
  return env.CHANGE_BRANCH == null ? env.BRANCH_NAME : env.CHANGE_BRANCH
}

def getVersion(String libraryName) {
  def props = readJSON file: "libs/${libraryName}/package.json"
  return props.version
}

def isMain() {
  return env.BRANCH_NAME == "main"
}

def affected(String project) {
  return env.affected_libraries.split(', ').contains(project)
}
