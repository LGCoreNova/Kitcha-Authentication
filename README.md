## 서비스 개요
사용자 인증과 인가를 담당하는 마이크로서비스입니다.

회원 가입, 로그인, JWT 토큰 발급 및 검증 등의 기능을 제공합니다. 

유저 정보는 내부적으로 MySQL 데이터베이스에 저장되며, 다른 서비스에서 사용자 인증이 필요할 때 이 서비스가 발급한 토큰을 기반으로 검증을 수행합니다.
<br><br>

## 빌드 및 실행 방법(Jenkins + ECS + ECR 기반)
Jenkins를 활용한 자동화 빌드 및 배포 파이프라인과, AWS ECS 기반의 컨테이너 실행 및 관리 체계를 구성하였습니다.

빌드된 Docker 이미지는 latest 태그로 ECR에 푸시되며, ECS 서비스는 해당 태그를 기준으로 항상 최신 버전을 배포합니다.

일반적인 Jenkinsfile 파이프라인 예시는 다음과 같습니다.

### 📦 Jenkinsfile 예시

```bash
Pipeline {
  Tools: gradle 8.12.1

  Stages:
    1. Gradle Build
       - GitHub에서 코드 pull
       - gradle clean build -x test

    2. Docker Build & Deploy (조건: DOCKER_BUILD == true)
       - SSH로 Docker 서버 접속
       - Docker 로그인 (ECR)
       - 이미지 빌드 및 태깅
       - Docker 이미지 push (ECR)
       - ECS 태스크 정의 json 추출 → jq로 필터링
       - ECS 태스크 재등록
       - ECS 서비스에 새 태스크 배포
}
```
<br>

## 환경 변수 설정
ECS 태스크 정의 시 environment로 설정
- authentication-service의 ECS Task 정의 내 환경변수 예시
```json
"environment": [
    {
        "name": "MYSQL_DATABASE",
        "value": "example-database"
    },
    {
        "name": "MYSQL_PASSWORD",
        "value": "example-password"
    },
    {
        "name": "TOKEN_EXPIRATION_TIME",
        "value": "example-expiration-time"
    },
    {
        "name": "TEST_USER_PASSWORD",
        "value": "example-user-password"
    },
    {
        "name": "AUTH_HOSTNAME",
        "value": "example-auth-hostname"
    },
    {
        "name": "MYSQL_USER",
        "value": "example-username"
    },
    {
        "name": "TEST_ADMIN_PASSWORD",
        "value": "example-admin-password"
    },
    {
        "name": "TOKEN_SECRET",
        "value": "example-token-secret"
    }
]

```






