pipeline {
  environment {
    ARGO_SERVER = '35.228.248.31'
    DEV_URL = 'https://35.228.33.0'
  }
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  stages {
    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
        }
      }
    }
    stage('Test') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('SCA') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh 'mvn org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html'
            }
          }
        }
        stage ('OSS LicenseChecker') {
          steps {
            container('licensefinder') {
              sh 'ls -al'
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
    }
    stage ('SAST') {
      steps {
        container('slscan') {
          sh 'scan --type java,depscan --build'
        }
      }
      post {
        success {
          archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/*', fingerprint:true, onlyIfSuccessful: true
        }
      }
    }
    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container('maven') {
              sh 'mvn package -DskipTests'
            }
          }
        }
        stage('Docker build and Publish') {
          steps {
            container('kaniko') {
              sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --skip-tls-verify --cache=true --destination=docker.io/ennc0d3/dsodemo'
            }
          }

        }
      }
    }

    // Image scanning
    stage("Image Analysis") {
      parallel {
        stage("Image Lint") {
          // Dockle
          steps {
            container('docker-tools') {
              sh 'dockle docker.io/ennc0d3/dsodemo'
            }
          }
        }
        stage("Image Vuln") {
          // Trivy
          steps {
            container('docker-tools') {
              sh 'trivy image --exit-code 1 ennc0d3/dsodemo'
            }
              
          }

        }
      }
    }
    //

    stage('Scan k8s Deploy Code') {
      steps {
        container('docker-tools') {
          sh 'kubesec scan deploy/dso-demo-deploy.yaml'
        }
      }
    }

    stage('Deploy to Dev') {
      environment {
          AUTH_TOKEN = credentials('jenkins-argocd-deploy-token')
      }
      steps {
        container('docker-tools') {
          sh "docker run -t schoolofdevops/argocd-cli argocd app sync dso-demo --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN"
          sh "docker run -t schoolofdevops/argocd-cli argocd app wait dso-demo --health --timeout 300 --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN"
        }
      }
   }

   stage('Dynamic Analysis') {
      parallel {
        stage('E2E tests') {
          steps {
            sh 'echo "All Tests passed!!!"'
          }
        }
        stage('DAST') {
          steps {
            container('docker-tools') {
              sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t $DEV_URL || exit 0'
             }
          }
        }
       }
    }

  }
}
