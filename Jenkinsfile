def APP =  env.JOB_NAME.split('/').last().toLowerCase()
def COMMITID = ""
def TIMESTAMP = ""
properties([
    parameters([
        choice(
            choices: ['git@github.com:yjiangi/ci-demo.git'], 
            description: '选择服务', 
            name: 'GIT_URL'
        ),
        reactiveChoice(
            choiceType: 'PT_SINGLE_SELECT', 
            description: '选择分支', 
            filterLength: 0, 
            filterable: false, 
            name: 'BRANCH', 
            randomName: 'choice-parameter-${UUID.randomUUID().toString().substring(0, 4)}', 
            referencedParameters: 'GIT_URL', 
            script: groovyScript(
                fallbackScript: [
                    classpath: [], 
                    oldScript: '', 
                    sandbox: true, 
                    script: 'return [""]'
                ], 
                script: [
                    classpath: [], 
                    oldScript: '', 
                    sandbox: true, 
                    script: '''                   
def getTags = "git ls-remote --heads git@github.com:yjiangi/ci-demo.git".execute()
return getTags.text.readLines().collect { it.split()[1].replaceAll('refs/heads/', '') }.unique()
                    '''
                ]
            )
        )
    ])
])
pipeline {
    agent {
        kubernetes {
            label "jnlp-slave-${UUID.randomUUID().toString().substring(0, 8)}"
            yamlFile "ci/jnlp.yaml"
        }
    }
    environment {
        DOCKER_REGISTRY = "registry.cn-hangzhou.aliyuncs.com"
        REGISTRY_NAMEPSACE = "gitops-demo"
        IMAGE = "${DOCKER_REGISTRY}/${REGISTRY_NAMEPSACE}"
    }
    options {
        //保持构建15天 最大保持构建的30个 发布包保留15天
        buildDiscarder logRotator(artifactDaysToKeepStr: '15', artifactNumToKeepStr: '', daysToKeepStr: '15', numToKeepStr: '30')
        //时间模块
        timestamps()
        //超时时间
        timeout(time:60, unit:'MINUTES')
    }

    stages {
       stage('pull code') {
            steps {                          
                echo '--------------------------拉取代码-----------------------'
                checkout([$class: 'GitSCM', branches: [[name: "${BRANCH}"]], 
                extensions: [], userRemoteConfigs: [[credentialsId: 'github-ci', 
                url: "${GIT_URL}"]]])
            }
        }           
        stage('commit'){
            steps{
              script{
                  COMMITID = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                  TIMESTAMP = sh(script: "date +%Y%m%d%H%M-%S", returnStdout: true).trim()
                  env.ImageTag = "${BUILD_ID}-${TIMESTAMP}-${COMMITID}"
                  env.AppName =  env.JOB_NAME.split('/').last().toLowerCase()
                  sh """
                  echo "仓库地址: ${GIT_URL}"
                  echo "服务分支: ${BRANCH}"
                  echo "分支id: ${COMMITID}"
                  echo "构建时间: ${TIMESTAMP}"
                  echo "镜像TAG: ${ImageTag}"
                  echo "服务名字: ${AppName}"
                  echo "服务名称: ${APP}"
                  """
                }   
            }
        }
        stage('build image') {
            steps {
                container('docker') {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding',
                        credentialsId: 'docker-auth',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD']]) {
                          script {
                            sh """
                            
                            echo "开启多架构编译"
                            docker buildx create --name mybuilder --use --driver docker-container --driver-opt image=registry.cn-hangzhou.aliyuncs.com/s-ops/buildkit:buildx-stable-1

                            echo "登陆仓库"
                            docker login ${DOCKER_REGISTRY} -u ${DOCKER_USER} -p ${DOCKER_PASSWORD}

                            echo "构建/推送镜像"
                            docker buildx build --progress=plain --no-cache --platform=linux/amd64 -f Dockerfile -t ${IMAGE}/${AppName}:${ImageTag} . --push

                            """
                        }
                    }    
                }
            }
        }
      stage('change ImageTag') {
          steps {
                container('tools'){
                    script{
                      sh """
                       envsubst < ./values.tpl > helm/values.yaml
                       cat helm/values.yaml
                       helm template --debug  helm/  -f helm/values.yaml
                      """
                    }
                }
            }
        }
        stage('push yaml') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'github-ci', keyFileVariable: 'IDENTITY')]) {
                    script {
                      sh """
                        git config --global user.email "ci"
                        git config --global user.email "ci@ci.com"
                        git config core.sshCommand 'ssh -o StrictHostKeyChecking=no -i $IDENTITY'
                        git checkout main
                        git pull origin main
                        git add .
                        git commit -m "${AppName}-${ImageTag} " || true
                        git push origin main  
                        """
                    }    
                }
            }
        }       
    }
}
