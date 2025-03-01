pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  environment {
    APP_NAME = "sample-api-service"
    IMAGE_REGISTRY = "rmkanda"
  }
  stages {
    stage('Setup') {
      parallel {
        stage('Install Dependencies') {
          steps {
            container('maven') {
              sh './mvnw install -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
            }
          }
        }
        stage('Secrets scanner') {
          steps {
            container('trufflehog') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh "trufflehog -x secrets-exclude.txt ${GIT_URL}"
              }
            }
          }
        }
      }
    }
    stage('Build') {
      steps {
        container('maven') {
          sh './mvnw package -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
        }
      }
    }
    // stage('SCA') {
    //   steps {
    //     container('maven') {
    //       sh './mvnw org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
    //     }
    //   }
    //   post {
    //     success {
    //       dependencyTrackPublisher projectName: 'sample-spring-app', projectVersion: '0.0.1', artifact: 'target/bom.xml', autoCreateProjects: true, synchronous: true
    //       archiveArtifacts allowEmptyArchive: true, artifacts: 'target/bom.xml', fingerprint: true, onlyIfSuccessful: true
    //     }
    //   }
    // }
    stage('Static Analysis') {
      parallel {
        stage('OSS License Checker') {
          steps {
            container('licensefinder') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh '''#!/bin/bash --login
                      /bin/bash --login
                      rvm use default
                      gem install license_finder
                      license_finder
                    '''
              }
            }
          }
        }
        stage('Spot Bugs - Security') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh './mvnw compile spotbugs:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/spotbugsXml.xml', fingerprint: true, onlyIfSuccessful: false
              recordIssues enabledForFailure: true, tool: spotBugs()
            }
          }
        }
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh './mvnw test'
            }
          }
        }
        stage('Dependency Check') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh './mvnw org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: false
              dependencyCheckPublisher pattern: 'target/dependency-check-report:xml'
            }
          }
        }
      }
    }
    stage('Package') {
      steps {
        container('docker-tools') {
          sh "docker build . -t ${APP_NAME}"
        }
      }
    }
    stage('Artifact Analysis') {
      parallel {
        stage('Container Scan') {
          steps {
            container('docker-tools') {
              sh "grype ${APP_NAME}"
            }
          }
        }
        stage('Container Audit') {
          steps {
            container('docker-tools') {
              echo "dockle ${APP_NAME}"
              // sh "dockle ${APP_NAME}"
            }
          }
        }
        stage('Kubesec') {
          steps {
            container('docker-tools') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh "kubesec scan k8s.yaml"
              }
            }
          }
        }
      }
    }
    stage('Publish') {
      steps {
        container('docker-tools') {
          echo "Publishing docker image"
          // sh "docker push ${APP_NAME}"
        }
      }
    }
    stage('Deploy to Dev') {
      steps {
        container('docker-tools') {
          echo "Deploying the app"
          // sh "kubectl apply -f k8s.yaml"
        }
      }
    }
    stage('DAST') {
      steps {
        container('docker-tools') {
          sh "docker pull owasp/zap2docker-weekly"
          sh "docker run -t owasp/zap2docker-weekly zap-api-scan.py -t http://172.17.0.3:30001/v3/api-docs -f openapi"
          // sh "docker push ${APP_NAME}"
        }
      }
    }
    stage('Promote to Prod') {
      steps {
        container('docker-tools') {
          echo "Promote to Prod"
        }
      }
    }
  }
}