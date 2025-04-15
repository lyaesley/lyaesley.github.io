---
title: "Spring WebFlux에서 403 상태 처리와 API Key 갱신"
category: [pipeline]
tag: [github-action, GCP, docker]
toc: true
toc_sticky: true
published: true
---

## GitHub Actions를 활용한 CI/CD 파이프라인 구축과 최신 Docker 이미지 배포까지

GitHub Actions를 활용하여 CI/CD 파이프라인을 설계하고, Docker 기반의 애플리케이션 이미지를 GitHub Packages에 푸시한 뒤 Google Cloud Platform(GCP) VM으로 배포하는 작업을 어떻게 구성했는지 공유하려고 합니다.

---

### **파이프라인 구성 목표**

CI/CD 파이프라인은 다음을 목표로 설계했습니다:
1. `main` 브랜치에 코드가 푸시되거나 PR(Pull Request)이 생성될 때 자동 실행.
2. **Java 애플리케이션 빌드 및 테스트**, 이후 Docker 이미지를 생성하고 GitHub Container Registry(ghcr.io)에 푸시.
3. 푸시된 이미지를 **GCP VM**으로 가져와 실행.

---

### **GitHub Actions Workflow 살펴보기**

#### 1. **파이프라인 실행 조건**
아래 조건에 따라 워크플로가 트리거됩니다:
- `main` 브랜치로 **push** 또는 **pull_request** 시 실행.
- 수동 실행이 필요한 경우 `workflow_dispatch` 이벤트로 실행 가능.

```yaml
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:
```

---

#### 2. **환경 변수 정의 및 저장소 이름 변환**
여기에서 GitHub Actions 표현식으로 환경 변수(`REGISTRY`)를 정의합니다. 저장소 이름(`github.repository_owner`와 `github.event.repository.name`)은 기본적으로 대소문자를 구분하므로, 이를 전환하기 위해 Bash 명령어를 사용했습니다.

```yaml
- name: Set lowercase image name
  run: |
    echo "IMAGE_NAME=${OWNER,,}/${REPO,,}" >> ${GITHUB_ENV}
  env:
    OWNER: ${{ github.repository_owner }}
    REPO: ${{ github.event.repository.name }}
```

이 코드는 저장소의 소유자와 이름을 소문자로 변환해 `IMAGE_NAME` 변수에 저장합니다. 모든 단계에서 이를 재사용하여 Docker 이미지 이름을 동적으로 설정할 수 있습니다.

---

#### 3. **Java 빌드와 캐시 설정**
워크플로는 Java 17을 사용하며, Gradle 빌드 캐싱을 적용합니다. 다음 단계에 따라 빌드를 진행합니다:
1. 애플리케이션의 코드를 체크아웃.
2. Java 17(Temurin) 환경 설치.
3. Gradle 캐시를 활성화하여 빌드 시간 단축.

```yaml
- name: Set up Java 17
  uses: actions/setup-java@v3
  with:
    distribution: 'temurin'
    java-version: '17'

- name: Cache Gradle packages
  uses: actions/cache@v3
  with:
    path: ~/.gradle/caches
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: |
      ${{ runner.os }}-gradle-
```

Gradle 빌드 단계에서는 다음과 같이 테스트를 제외하고 클린 빌드를 수행합니다:
```yaml
- name: Build Java Application
  run: ./gradlew clean build -x test
```

---

#### 4. **Docker 이미지 빌드 및 GitHub Packages 푸시**
이미지를 빌드하고 저장소에 푸시하는 작업은 다음과 같이 구성됩니다:

**(1) 메타데이터 추출**
Docker 이미지에 필요한 태그와 라벨 정보를 추출합니다:
```yaml
- name: Extract metadata (tags, labels) for Docker
  id: meta
  uses: docker/metadata-action@v4
  with:
    images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
```

**(2) Docker 이미지 빌드와 푸시**
`docker/build-push-action`을 사용해 이미지를 빌드하고, GitHub Packages로 푸시합니다:
```yaml
- name: Build and Push Docker Image
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./App.Dockerfile
    push: true
    tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```
`latest` 태그를 포함하여 항상 최신 이미지를 푸시합니다. 이를 통해 배포 환경에서는 간단히 `latest` 태그로 이미지를 참조하도록 구성할 수 있습니다.

---

#### 5. **GCP VM으로 배포**
GCP VM 환경에 SSH로 연결하여 배포를 진행합니다:
1. GCP의 Docker 환경에 로그인.
2. 기존 컨테이너(서비스)가 실행 중이면 중단 및 삭제.
3. GitHub Packages에서 push된 최신 이미지를 pull.
4. 가져온 이미지를 기반으로 새로운 컨테이너 실행.

**배포 스크립트**는 다음과 같습니다:
```yaml
- name: Deploy to GCP VM via SSH Action
  uses: appleboy/ssh-action@master
  with:
    host: ${{ secrets.GCP_HOST }}
    username: ${{ secrets.GCP_USERNAME }}
    key: ${{ secrets.GCP_SSH_PRIVATE_KEY }}
    passphrase: ""  # (필요시 설정)
    script: |
      docker login ghcr.io -u ${{ github.actor }} -p ${{ secrets.GITHUB_TOKEN }}
      docker stop notice-service || true
      docker rm notice-service || true
      docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
      docker run -d --name notice-service -p 8080:8080 ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```

이 배포 과정은 `latest` 태그가 항상 최신 이미지를 가리키도록 보장합니다.

---

### **`latest` 태그와 관리 전략**

`latest` 태그는 Docker 컨테이너에서 특정 버전이 아닌 최신 이미지를 가져오도록 하는 데 유용합니다. 이번 프로젝트에서도 배포 과정에서 간소화된 참조를 위해 기본적으로 `latest`를 활용했습니다.

다만, **최신 안정 버전에 대한 보장이 필요**한 경우, GitHub Actions에서 CI/CD 실행 번호(`github.run_number` 등)를 활용해 새로운 버전 태그를 추가로 관리할 수도 있습니다.

```yaml
tags: |
  ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
  ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.run_number }}
```

`latest`는 간편하지만, 운영 환경에 따라 버전 태깅 전략을 병행하여 관리하는 것이 좋습니다.

---

### **마무리**

이번 게시글에서는 GitHub Actions와 Docker를 활용해 CI/CD 파이프라인을 구성하고, 최신 이미지를 GCP VM으로 성공적으로 배포하는 과정을 자세히 살펴봤습니다. 위 과정을 통해 반복적인 배포를 자동화하고 최신 코드를 안정적으로 운영 환경에 반영할 수 있었습니다.

매번 수작업으로 배포하던 과정을 이렇게 단순화할 수 있다는 점은 정말 큰 장점입니다. 여러분의 프로젝트에도 적용해 보시고, 필요하신 부분이 있다면 언제든 질문해주세요! 😊

--- 

이 블로그 글이 여러분의 CI/CD 활용에 도움이 되길 바랍니다! 🚀