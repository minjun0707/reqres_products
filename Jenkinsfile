pipeline {
    agent any

    environment {
        REGISTRY = 'user04acr.azurecr.io'
        IMAGE_NAME = 'product'
        AKS_CLUSTER = 'user04-aks'
        RESOURCE_GROUP = 'user04-rsrcgrp'
        AKS_NAMESPACE = 'default'
        AZURE_CREDENTIALS_ID = 'Azure-Cred'
        TENANT_ID = 'f46af6a3-e73f-4ab2-a1f7-f33919eda5ac' // Service Principal 등록 후 생성된 ID
        GITHUB_REPO = "github.com/minjun0707/reqres_products.git"
    }
 
    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }
        
        stage('Maven Build') {
            steps {
                withMaven(maven: 'Maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    image = docker.build("${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Azure Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.AZURE_CREDENTIALS_ID, usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                        sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant ${TENANT_ID}'
                    }
                }
            }
        }
        
        stage('Push to ACR') {
            steps {
                script {
                    sh "az acr login --name ${REGISTRY.split('\\.')[0]}"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('CleanUp Images') {
            steps {
                sh """
                docker rmi ${REGISTRY}/${IMAGE_NAME}:v$BUILD_NUMBER
                """
            }
        }
        
        stage('Deploy to AKS') {
            steps {
                script {
                    sh "az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${AKS_CLUSTER}"
                    sh """
                    sed 's/latest/v${env.BUILD_ID}/g' kubernetes/deploy.yaml > output.yaml
                    cat output.yaml
                    kubectl apply -f output.yaml
                    kubectl apply -f kubernetes/service.yaml
                    rm output.yaml
                    """
                }
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
            withCredentials([usernamePassword(credentialsId: 'Github-Cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                sh """
                    git config --global user.email "jmk7117@naver.com"
                    git config --global user.name "minjun0707"

                    # GitHub 저장소 클론 (PAT 사용)
                    git clone https://${GIT_USERNAME}:${GIT_TOKEN}@${GITHUB_REPO} repo

                    # deploy.yaml 파일 복사
                    cp kubernetes/deploy.yaml repo/kubernetes/deploy.yaml

                    # 변경 사항 커밋 및 푸시
                    cd repo
                    git add kubernetes/deploy.yaml
                    git commit -m "Update deploy.yaml with build ${env.BUILD_NUMBER}"
                    git push https://${GIT_USERNAME}:${GIT_TOKEN}@${GITHUB_REPO} master

                    # 작업 폴더 정리
                    cd ..
                    rm -rf repo
                """
            }
        }
    }
}
 







        
    }
}
