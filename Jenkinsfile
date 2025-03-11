pipeline { agent any

environment {
    REGISTRY = 'user04acr.azurecr.io'
    IMAGE_NAME = 'product'
    AKS_CLUSTER = 'user04-aks'
    RESOURCE_GROUP = 'user04-rsrcgrp'
    AKS_NAMESPACE = 'default'
    AZURE_CREDENTIALS_ID = 'Azure-Cred'
    TENANT_ID = 'f46af6a3-e73f-4ab2-a1f7-f33919eda5ac' // Service Principal 등록 후 생성된 ID
    GIT_USER_NAME = 'minjun0707'
    GIT_USER_EMAIL = 'jmk7117@naver.com'
    GITHUB_CREDENTIALS_ID = 'Github-Cred'
    GITHUB_REPO = 'github.com/minjun0707/reqres_products.git'
    GITHUB_BRANCH = 'master' // 업로드할 브랜치
}

    stages {
        stage('Check Modified Files') {
            steps {
                script {
                    def services = SERVICES.tokenize(',') // Use tokenize to split the string into a list
                    for (int i = 0; i < services.size(); i++) {
                        def service = services[i] // Define service as a def to ensure serialization
                        // Git 변경된 파일 목록 가져오기
                        def changedFiles = sh(returnStdout: true, script: 'git diff --name-only HEAD~1 HEAD').trim().split('\n')

                    // 'src/' 폴더 변경 여부 확인
                        def targetFolder = 'order/src/*'
                        def isModified = changedFiles.any { it.startsWith(targetFolder) }
        
                        if (isModified) {
                            echo "Changes detected in ${targetFolder}. Proceeding with the pipeline."
                        } else {
                            echo "No changes in ${targetFolder}. Stopping pipeline execution."
                            currentBuild.result = 'NOT_BUILT'
                            return
                        }
                    }
                }
            }
        }
        stage('Clone Repository') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                checkout scm
            }
        }
        
        stage('Maven Build') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                withMaven(maven: 'Maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        
        stage('Docker Build') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    image = docker.build("${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Push to ACR') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    sh "az acr login --name ${REGISTRY.split('\\.')[0]}"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}"
            
                }
            }
        }

    
        
        stage('CleanUp Images') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                sh """
                docker rmi ${REGISTRY}/${IMAGE_NAME}:v$BUILD_NUMBER
                """
            }
        }
        
        stage('Update deploy.yaml') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    sh """
                    sed -i 's|image: \"${REGISTRY}/${IMAGE_NAME}:.*\"|image: \"${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_ID}\"|' kubernetes/deploy.yaml
                    cat kubernetes/deploy.yaml
                    """
                }
            }
        }
        
        stage('Commit and Push to GitHub') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GITHUB_CREDENTIALS_ID, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh """
                            git config --global user.email "jmk7117@naver.com"
                            git config --global user.name "Jenkins CI"
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${GITHUB_REPO} repo
                            cp kubernetes/deploy.yaml repo/kubernetes/deploy.yaml
                            cd repo
                            git add kubernetes/deploy.yaml
                            git commit -m "Update deploy.yaml with build ${env.BUILD_NUMBER}"
                            git push origin ${GITHUB_BRANCH}
                            cd ..
                            rm -rf repo
                        """
                    }
                }
            }
        } 
    }
}
