def namespace = "production"
def serviceName = "jobber-users"
def service = "Jobber Users"

def groovyMethods

def m1 = System.currentTimeMillis()

pipeline {
  agent {
    label 'jenkins-agent'
  }

  tools {
    nodejs "NodeJS"
    dockerTool "Docker"
  }

  environment {
    DOCKER_CREDENTIALS = credentials("dockerhub")
    NPM_TOKEN = credentials("github-npm-token")
    IMAGE_NAME = "kevin1208" + "/" + "jobber-users"
    IMAGE_TAG = "stable-${BUILD_NUMBER}"
  }

  stages {
    stage("Cleanup Workspace") {
      steps {
        cleanWs()
      }
    }

    stage("Prepare Environment") {
      steps {
        sh "[ -d pipeline ] || mkdir pipeline"
        dir("pipeline") {
          // Add your jenkins automation url to url field
          git branch: 'main', credentialsId: 'github', url: 'https://github.com/kevindeveloper95/jenkins-automation.git'
          script {
            groovyMethods = load("functions.groovy")
          }
        }
        // Add your users service url to url field
        git branch: 'master', credentialsId: 'github', url: 'https://github.com/kevindeveloper95/jobapp-users'
        script {
          // Configure npm for private packages
          sh '''
            echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" > .npmrc
            echo "@kevindeveloper95:registry=https://npm.pkg.github.com" >> .npmrc
          '''
          // Try npm first, fallback to Docker if it fails
          try {
            sh 'npm install'
          } catch (Exception e) {
            echo "npm failed, using Docker fallback..."
            sh '''
              docker run --rm -v $(pwd):/app -w /app \
              -e NPM_TOKEN=$NPM_TOKEN \
              node:18-alpine sh -c "
                echo '//npm.pkg.github.com/:_authToken='$NPM_TOKEN > .npmrc &&
                echo '@kevindeveloper95:registry=https://npm.pkg.github.com' >> .npmrc &&
                npm install
              "
            '''
          }
        }
      }
    }

     stage("Lint Check") {
      steps {
        sh 'npm run lint:check'
      }
    }

    stage("Code Format Fix") {
      steps {
        sh '''
          # Fix formatting automatically
          npm run prettier:fix || echo "Prettier fix completed"
          # Check if there are still issues
          npm run prettier:check
        '''
      }
    }

    stage("Unit Test") {
      steps {
        sh 'npm run test'
      }
    }

    stage("Build and Push") {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh "docker login -u ${DOCKER_USER} --password ${DOCKER_PASS}"
            sh "docker build -t $IMAGE_NAME ."
            sh "docker tag $IMAGE_NAME $IMAGE_NAME:$IMAGE_TAG"
            sh "docker tag $IMAGE_NAME $IMAGE_NAME:stable"
            sh "docker push $IMAGE_NAME:$IMAGE_TAG"
            sh "docker push $IMAGE_NAME:stable"
          }
        }
      }
    }

    stage("Clean Artifacts") {
      steps {
        sh "docker rmi $IMAGE_NAME:$IMAGE_TAG"
        sh "docker rmi $IMAGE_NAME:stable"
      }
    }

    stage("Create New Pods") {
      steps {
        withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'minikube', contextName: 'minikube', credentialsId: 'jenkins-k8s-token', namespace: '', serverUrl: 'https://host.docker.internal:50470']]) {
          script {
            def pods = groovyMethods.findPodsFromName("${namespace}", "${serviceName}")
            for (podName in pods) {
              sh """
                kubectl delete -n ${namespace} pod ${podName}
                sleep 10s
              """
            }
          }
        }
      }
    }
  }
  post {
    success {
      script {
        def m2 = System.currentTimeMillis()
        def durTime = groovyMethods.durationTime(m1, m2)
        def author = groovyMethods.readCommitAuthor()
        groovyMethods.notifySlack("", "jobber-jenkins", [
        				[
        					title: "BUILD SUCCEEDED: ${service} Service with build number ${env.BUILD_NUMBER}",
        					title_link: "${env.BUILD_URL}",
        					color: "good",
        					text: "Created by: ${author}",
        					"mrkdwn_in": ["fields"],
        					fields: [
        						[
        							title: "Duration Time",
        							value: "${durTime}",
        							short: true
        						],
        						[
        							title: "Stage Name",
        							value: "Production",
        							short: true
        						],
        					]
        				]
        		]
        )
      }
    }
    failure {
      script {
        def m2 = System.currentTimeMillis()
        def durTime = groovyMethods?.durationTime(m1, m2) ?: "Unknown"
        def author = groovyMethods?.readCommitAuthor() ?: "Unknown"
        if (groovyMethods) {
          groovyMethods.notifySlack("", "jobber-jenkins", [
        				[
        					title: "BUILD FAILED: ${service} Service with build number ${env.BUILD_NUMBER}",
        					title_link: "${env.BUILD_URL}",
        					color: "error",
        					text: "Created by: ${author}",
        					"mrkdwn_in": ["fields"],
        					fields: [
        						[
        							title: "Duration Time",
        							value: "${durTime}",
        							short: true
        						],
        						[
        							title: "Stage Name",
        							value: "Production",
        							short: true
        						],
        					]
        				]
        		]
          )
        } else {
          echo "groovyMethods not available, skipping Slack notification"
        }
      }
    }
  }
}
