pipeline {
  agent none
  
  tools {
      gradle "gradle8.12.1"
  }
  
  parameters {
      booleanParam(name: 'DOCKER_BUILD', defaultValue: true, description: 'Docker 이미지 빌드 실행 여부')
      string(name: 'DOCKER_IMAGE_TAG', defaultValue: '', description: 'Docker 이미지 태그 (비워두면 빌드 번호 사용)')
  }
  
  stages {
    stage('Gradle Install') {
      agent any
      steps {
        git branch: 'main',
        url: 'https://github.com/LGCoreNova/Kitcha-Authentication.git'
        sh 'gradle clean build -x test'
      }
    }
    
    stage('Docker Image Build') {
      agent any
      when {
        expression { params.DOCKER_BUILD == true }
      }
      steps {
          script {
            if (params.DOCKER_IMAGE_TAG != "") {
                echo "[docker image tag is not null]"
                imageTag = params.DOCKER_IMAGE_TAG
            } else {
                echo "[docker image tag is null]"
                imageTag = env.BUILD_NUMBER
            }
          }
          
          sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'toy-msa-docker',
                        transfers: [sshTransfer(
                            cleanRemote: false,
                            excludes: '',
                            execCommand: 'cd kitcha/auth && docker build --tag auth:' + imageTag + ' -f Dockerfile .',
                            execTimeout: 600000,
                            flatten: false,
                            makeEmptyDirs: false,
                            noDefaultExcludes: false,
                            patternSeparator: '[, ]+',
                            remoteDirectory: './kitcha/auth',
                            remoteDirectorySDF: false,
                            removePrefix: 'build/libs',
                            sourceFiles: 'build/libs/*.jar'
                        )],
                        usePromotionTimestamp: false,
                        useWorkspaceInPromotion: false,
                        verbose: false
                    )
          ])
      }
    }
    
    stage('Deploy Service') {
      agent any
      when {
        expression { params.DOCKER_BUILD == true }
      }
      steps {
          sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'toy-msa-docker',
                        transfers: [sshTransfer(
                            cleanRemote: false,
                            excludes: '',
                            execCommand: '''
                                cd kitcha
                                docker stop auth || true
                                docker rm auth || true
                                docker run -d --name auth \\
                                  --network kitcha_network \\
                                  -e MYSQL_DATABASE=kitchadb \\
                                  -e MYSQL_USER=kitcha \\
                                  -e MYSQL_PASSWORD=p@ssw0rd \\
                                  -e CONFIG_SERVER_URI=http://config-server:8071 \\
                                  -p 8090:8080 \\
                                  auth:''' + imageTag,
                            execTimeout: 180000,
                            flatten: false,
                            makeEmptyDirs: false,
                            noDefaultExcludes: false,
                            patternSeparator: '[, ]+',
                            remoteDirectory: '',
                            remoteDirectorySDF: false,
                            removePrefix: '',
                            sourceFiles: ''
                        )],
                        usePromotionTimestamp: false,
                        useWorkspaceInPromotion: false,
                        verbose: true
                    )
          ])
      }
    }
    
    stage('Verify Deployment') {
      agent any
      when {
        expression { params.DOCKER_BUILD == true }
      }
      steps {
          sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'toy-msa-docker',
                        transfers: [sshTransfer(
                            cleanRemote: false,
                            excludes: '',
                            execCommand: '''
                                # 서비스 상태 확인
                                sleep 10
                                if docker logs auth | grep "Started AuthenticationApplication"; then
                                  echo "AUTH 서비스가 성공적으로 시작되었습니다."
                                  exit 0
                                else
                                  echo "AUTH 서비스 시작 실패!"
                                  exit 1
                                fi
                            ''',
                            execTimeout: 60000,
                            flatten: false,
                            makeEmptyDirs: false,
                            noDefaultExcludes: false,
                            patternSeparator: '[, ]+',
                            remoteDirectory: '',
                            remoteDirectorySDF: false,
                            removePrefix: '',
                            sourceFiles: ''
                        )],
                        usePromotionTimestamp: false,
                        useWorkspaceInPromotion: false,
                        verbose: true
                    )
          ])
      }
    }
  }
  
  post {
    success {
      echo "빌드 및 배포 성공!"
    }
    failure {
      echo "빌드 또는 배포 실패!"
    }
    always {
      cleanWs()
    }
  }
}