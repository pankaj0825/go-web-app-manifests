pipeline {
    agent any
    environment {
        GOLANGCI_LINT_VERSION = 'v1.56.2'
        docker_tag = gitVersion().trim()
    }
    
    tools { 
        go 'go-1.22.5'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/pankaj0825/go-web-app.git']])
            }
        }    
        stage('Build') {
            steps {
                sh 'go build -o go-web-app'
            }
        }
        stage('Test') {
            steps {
                echo 'go test ./...'
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }
        stage('Install golangci-lint') {
            steps {
                withEnv(["PATH+GO=/var/lib/jenkins/go/bin"]) {
                    sh """
                        curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b \$(go env GOPATH)/bin ${GOLANGCI_LINT_VERSION}
                        golangci-lint --version
                    """
                }    
            }
        }
        stage('Code Quality') {
            steps {
                withEnv(["PATH+GO=/var/lib/jenkins/go/bin"]) {
                    sh "golangci-lint run"
                }
            }
        }
        stage('Docker Build and Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub') {
                    sh 'docker build -t pankaj011/go-web-app:${docker_tag} .'
                    }
                }   
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --format table -o image.html pankaj011/go-web-app:${docker_tag}'
            }
        }
        stage('Image Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub') {
                    sh 'docker push pankaj011/go-web-app:${docker_tag}'
                    }
                }   
            }
        }
        stage('Update Helm Chart') {
        /*    environment {
                GITHUB_TOKEN = credentials('go-web-app-devops-manifest')
            }
        */
            steps {
                script {
        /*            sh '''
                    git config --global user.email "pankajtripathi892@gmail.com"
                    git config --global user.name "Pankaj"
                    git remote set-url origin https://${GITHUB_TOKEN}@github.com/pankaj0825/go-web-app-manifests.git
                    '''
                    // Update the Helm chart with the new image tag
        */            
                    sh "sed -i 's/tag: .*/tag: \"${docker_tag}\"/' helm/go-web-app-charts/values.yaml"
                    
                    // Commit and push the changes
        /*            sh '''
                        git add helm/go-web-app-charts/values.yaml
                        git commit -m "Update tag in Helm chart"
                        git push origin main
                    '''
        */        }
            }
        }
        stage('Push Helm Directory to Separate Repo') {
           environment {
                GITHUB_TOKEN = credentials('go-web-app-devops-manifest')
            }
            steps {
                script {
                    sh 'rm -rf go-web-app-manifests'
                    sh 'git clone https://${GITHUB_TOKEN}@github.com/pankaj0825/go-web-app-manifests.git'
                    sh '''
                    cd go-web-app-manifests/
                    pwd
                    ls 
                    tree
                    sed -i "s/tag: .*/tag: \\"${docker_tag}\\"/" helm/go-web-app-charts/values.yaml
                    git config --global user.email "pankajtripathi892@gmail.com"
                    git config --global user.name "Pankaj"
                    git add .
                    git commit -m "Update Helm manifests"
                    git push origin main
                    '''
                }
            }
        }
    }
}

def gitVersion() {
    def commitHash = sh returnStdout: true, script: 'git rev-parse --short HEAD'
    return commitHash
}