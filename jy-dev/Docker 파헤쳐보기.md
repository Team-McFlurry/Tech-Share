# Docker 파헤쳐보기 - 1

# Docker는 무엇일까?

> Docker is an open platform for developing, shipping, and running applications. Docker enables you to separate your applications from your infrastructure so you can deliver software quickly.

공식문서에 따르면 Docker는 어플리케이션을 개발, 배포 실행하는 개방 플랫폼이라고 설명하고 있습니다. 그리고 사용하고있는 인프라 요소와 분리되어 어플리케이션을 제공할 수 있다고 합니다.

어떻게 인프라 요소와 분리되어 어플리케이션을 제공할 수 있는 것일까요? 한번 알아보도록 하겠습니다.

# Docker 아키텍처

![출처 : Docker 공식문서](https://github.com/user-attachments/assets/a69191be-4212-4184-b764-06cb22be0de2)

출처 : Docker 공식문서

도커 아키텍처를 보면 Docker는 Client-Server 아키텍처를 사용합니다. Client가 Docker Host로 요청을 하고 Docker Host가 복잡한 작업들을 수행합니다.

Docker 아키텍처에서 소개하는 여러 요소중에 필수적으로 알아야 할 Client ,Docker daemon, images, container, Registry에 대해 알아보도록 하겠습니다.

## Docker Daemon

Docker Daemon은 Client-Server 아키텍처에서 Server 역할을 담당합니다.

Docker API를 제공해 요청을 수신하고 Client가 원하는 동작(이미지 빌드, 이미지 배포, 이미지 실행 등)을 수행합니다. 또한 이미지, 컨테이너, 네트워크, 볼륨 관리 혹은 다른 docker daemon과 통신을 수행합니다.

Docker Daemon은 Client랑 같은 System일 수 있습니다.

## Client

Client는 간단하게 말하면 사용자입니다.

Client는 Docker Daemon을 통해 Docker API를 실행할 수 있습니다.

## Image

Image는 Docker Container가 어떻게 Container를 생성해야 하는지에 대한 읽기전용 템플릿입니다.

예를 들면 내가 ubuntu 운영체제 이미지를 base로 설정하고 그 안에 nginx나 java를 설치하는 과정을 image로 저장할 수 있습니다. 이는 한 이미지에서 다른 이미지를 base로 하는 경우가 많을 수 있다는 걸 의미합니다.

## Container

Container는 실행할 수 있는 image의 인스턴스를 의미합니다.

Container는 Host의 커널만 공유하고 다른 Container 혹은 Host의 프로세스와 완전히 격리됩니다. 그래서 별도의 영구저장소에 작업 내용을 저장하지 않으면 Container가 삭제되면 모든 작업 내용도 삭제됩니다.

> Host는 Docker Engine을 포함한 모든 소프트웨어가 실행되는 전체 시스템

어떻게 다른 Container 또는 Host의 프로세스와 격리되는지는 나중에 알아보도록 하겠습니다.

## Registry

Registry는 Docker Image 저장소라고 생각하시면 됩니다.

Docker를 사용할 때 docker pull 이라는 명령어를 통해 이미지를 가져오는 경우가 있는데 이때 Registry(Docker Hub 등등)을 통해 이미지를 저장소에서 가져올 수 있습니다.

# Docker를 왜 사용할까?

## 현대 소프트웨어의 발전

Docker가 상용화 된 배경을 살펴보면 현대 소프트웨어의 발전이 가장 큰 영향을 가져오게 되었다 할 수 있습니다. 이를 이해하기 위해 이전 소프트웨어 개발과 현대의 소프트웨어 개발에 대해 알아보도록 하겠습니다.

### 이전 소프트웨어 개발

- 온프레미스 환경이 일반적
- 모놀리식 아키텍처 어플리케이션이 일반적
- 개발, 테스트, 배포가 하나의 단위로 일반적으로 이루어짐

### 현대 소프트웨어 개발

- 클라우드 네이티브 환경 대중화
- 마이크로서비스 아키텍처 어플리케이션의 대중화
- DevOps 문화와 CI/CD 파이프라인의 대중화
- 반복적이고 점진적인 개발 방식을 강조하는 에자일 문화의 대중화

### VM(가상머신) 기반 인프라의 한계점

- VM은 각각 완전한 OS를 실행하므로 상당한 시스템 리소스(CPU, 메모리, 스토리지)를 소비
- VM은 프로비저닝 및 부팅 시간이 길어 CI/CD 파이프라인의 신속한 배포와 잦은 업데이트에 어려움
- VM은 이식성이 제한적임
  - 하드웨어 의존성이 존재(가상 하드웨어 생성시 필요)
  - 전체 OS, 어플리케이션 의존성, 사용자 데이터 등을 포함하기 때문에 이미지 크기가 큼

## Container 기반 인프라

Container 기반 인프라는 애플리케이션과 그 의존성을 격리된 환경에서 실행하는 가상화 기술을 사용한 인프라 구조입니다.

### 핵심 구성요소

- 컨테이너
  - 애플리케이션과 그 실행에 필요한 모든 의존성을 포함한 경량화된 패키지
- 컨테이너 런타임
  - 컨테이너를 실행하고 관리하는 소프트웨어 (예: Docker)
- 컨테이너 오케스트레이션 플랫폼
  - 다수의 컨테이너를 관리하고 조정하는 도구 (예: Kubernetes)

### 주요 특징

- 필요한 만큼의 리소스만 사용하여 하드웨어 활용도 극대화
- 각 컨테이너는 독립적으로 실행되어 컨테이너에 문제가 생기더라도 다른 컨테이너나 호스트 시스템에 영향을 주지 않음
- "Build Once, Run Anywhere" 원칙 실현을 통해 다양한 환경에서 일관되게 실행
  - 컨테이너 이미지는 한 번 빌드되면 변경되지 않음
- 호스트 OS 커널을 공유하여 오버헤드가 VM 기반 인프라보다 적음
- CI/CD 파이프라인을 더 효율적으로 만들어 줌
  - VM 기반 인프라보다 컨테이너는 빠르게 시작하고 중지할 수 있음
  - 환경 일관성이 높아 "works on my machine" 문제 감소
    - 이미지에 컨테이너 구동에 필요한 모든 의존성들이 포함되어 있음
  - 가벼운 이미지로 인해 저장 및 전송이 빠름
- Serverless 환경에 알맞음
- 이미지를 통한 쉬운 버전 관리
- 기존 애플리케이션을 컨테이너화하여 점진적인 현대화 가능
  - EX) 레거시를 클라우드 네이티브 환경으로 전환

# Docker 실습

가장 어려운 실습인 Hello World를 Docker로 출력하는 실습을 진행하도록 하겠습니다.

Container 기술은 OS 커널의 namespace와 cgroups같은 기술을 활용해 만든 기술이기 때문에 Container 생성에 필요한 기술을 지원하지 않는 Host OS인 경우 해당 기술을 지원하는 OS를 가진 VM이 필요합니다.

Docker 같은 경우는 Docker Desktop을 통해 Host OS가 Container 생성에 필요한 기술을 지원하지 않더라도 별도 VM으로 도커를 실행할 수 있는 필수적인 요소들을 제공해줍니다.

```bash
// Docker Registry에서 busybox 이미지 pull
// busybox는 매우 작고 경량화된 Linux 시스템을 제공하는 컨테이너 이미지
docker pull busybox

// Docker busybox로 Hello World 출력
docker run busybox echo "Hello World"
```

docker run [image] [명령어]는 명시한 image를 통해 컨테이너를 생성하고 컨테이너 내부에서 [명령어]를 실행합니다.

이때 컨테이너는 불변이라는 특징을 가지고 있기 때문에 이미지에 정의된 명령들을 순차적으로 실행하고 사용자가 입력한 [명령어]를 수행하면 exit(종료)되는 라이프사이클을 가지고 있습니다.

busybox의 컨테이너를 계속 실행상태로 유지하려면 아래와 같은 명령어를 통해 대화형 셸을 사용합니다.

```
docker run -it busybox sh
```

# 다음 내용

- 컨테이너는 어떤 원리로 동작할까?
