pipeline {
  agent any
  
  tools {
      gradle "gradle8.12.1"
  }
  
  parameters {
      booleanParam(name: 'DOCKER_BUILD', defaultValue: true, description: 'Docker 이미지 빌드 실행 여부')
      string(name: 'DOCKER_IMAGE_TAG', defaultValue: '', description: 'Docker 이미지 태그 (비워두면 빌드 번호 사용)')
  }
  
  stages {
    stage('Gradle Install') {
      steps {
        checkout scm
        sh 'gradle clean build -x test'
      }
    }
    
    stage('Docker Image Build') {
      when {
        expression { params.DOCKER_BUILD == true }
      }
      steps {
          script {
            def imageTag
            
            if (params.DOCKER_IMAGE_TAG != "") {
                imageTag = params.DOCKER_IMAGE_TAG
            } else {
                imageTag = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
            }
            
            def ecrTagPrefix = "803691999553.dkr.ecr.us-west-1.amazonaws.com/kitcha/auth"

            def deployTag = "latest"
            if (env.BRANCH_NAME != "main") {
                deployTag = env.BRANCH_NAME
            }

            sshPublisher(publishers: [
                sshPublisherDesc(
                    configName: 'toy-docker-server',
                    transfers: [sshTransfer(
                        cleanRemote: false,
                        excludes: '',
                        execCommand: """
                            cd kitcha/auth
                            aws ecr get-login-password --region us-west-1 | docker login --username AWS --password-stdin 803691999553.dkr.ecr.us-west-1.amazonaws.com
                            docker build --tag kitcha/auth:${imageTag} -f Dockerfile .
                            docker tag kitcha/auth:${imageTag} ${ecrTagPrefix}:${imageTag}
                            docker tag kitcha/auth:${imageTag} ${ecrTagPrefix}:${deployTag}
                            
                            
                            docker push ${ecrTagPrefix}:${deployTag}
                        """,
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
                    verbose: true
                )
            ])
          }
      }
    }
    
    stage('Deploy to Development') {
      when {
        expression { 
          return env.BRANCH_NAME.startsWith('feat-') || env.BRANCH_NAME == 'develop' 
        }
      }
      steps {
        echo "개발 환경에 배포 중: ${env.BRANCH_NAME} 브랜치"
      }
    }
    
    stage('Deploy to Production') {
      when {
        expression { 
          return env.BRANCH_NAME == 'main' 
        }
      }
      steps {
        echo "프로덕션 환경에 배포 중: main 브랜치"
        script {
          // 빌드 단계에서 사용한 이미지 태그를 배포에도 사용하기 위해 변수 설정
          def imageTag = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
          def deployTag = "latest"
          def ecrTagPrefix = "803691999553.dkr.ecr.us-west-1.amazonaws.com/kitcha/auth"
          
          sshPublisher(publishers: [
            sshPublisherDesc(
              configName: 'toy-docker-server',
              transfers: [sshTransfer(
                cleanRemote: false,
                excludes: '',
                execCommand: """
                  cd kitcha/auth
                  
                  # 사용할 이미지 확인
                  echo "배포 이미지: ${ecrTagPrefix}:${deployTag}"
                  
                  # ECS 배포 스크립트 실행
                  aws ecs describe-task-definition --task-definition kitcha-auth --output json > task-auth-definition.json
                  
                  # 컨테이너 이미지를 새 이미지로 명시적 업데이트
                  jq --arg IMG "${ecrTagPrefix}:${deployTag}" '.taskDefinition.containerDefinitions[0].image = \$IMG' task-auth-definition.json > task-auth-updated.json
                  
                  # 필수 필드만 추출
                  jq '{family: .taskDefinition.family, networkMode: .taskDefinition.networkMode, containerDefinitions: .taskDefinition.containerDefinitions, requiresCompatibilities: .taskDefinition.requiresCompatibilities, cpu: .taskDefinition.cpu, memory: .taskDefinition.memory, executionRoleArn: .taskDefinition.executionRoleArn, volumes: .taskDefinition.volumes, placementConstraints: .taskDefinition.placementConstraints}' task-auth-updated.json > clean-task-auth-def.json
                  
                  # 새 작업 정의 등록
                  TASK_DEF_ARN=\$(aws ecs register-task-definition --cli-input-json file://clean-task-auth-def.json --query 'taskDefinition.taskDefinitionArn' --output text)
                  
                  echo "생성된 작업 정의 ARN: \$TASK_DEF_ARN"
                  
                  # 새 작업 정의로 서비스 업데이트 - 서비스 이름 수정
                  aws ecs update-service --cluster LGCNS-Cluster-2 --service kitcha-auth-new-service --task-definition \$TASK_DEF_ARN --force-new-deployment
                  
                  echo "ECS 서비스 업데이트 완료: kitcha-auth-new-service (새 이미지, 강제 배포 적용)"
                """,
                execTimeout: 300000,
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
  }
  
  post {
    success {
      echo "빌드 및 배포 성공! 브랜치: ${env.BRANCH_NAME}"
    }
    failure {
      echo "빌드 또는 배포 실패! 브랜치: ${env.BRANCH_NAME}"
    }
    always {
      cleanWs()
    }
  }
}


