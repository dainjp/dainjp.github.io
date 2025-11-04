---
title: "Docker"
description: >-
  도커란 무엇인지, 
date: 2025-11-04 11:30:0 +0900
categories: [DevOps]
tags: [docker, development, container, tools]
---

## 1. Docker란 무엇인가?

"Docker는 무엇이든 어디서든, 똑같이 실행할 수 있게 해주는 컨테이너 플랫폼이다." - IBM

Docker에 대해 엄밀하지는 않지만 이해를 돕기 위해 한가지 비유로 예전 닌텐도 게임기를 떠올려보자. 게임 카트리지만 꽂으면 내 닌텐도든, 친구의 닌텐도든 어떤 게임기에서든 똑같이 게임이 실행됐다. 게임에 필요한 모든 것(그래픽, 사운드, 게임 로직)이 카트리지 안에 들어있었기 때문이다.

Docker 컨테이너도 이와 비슷하다. 애플리케이션과 실행에 필요한 모든 것을 하나로 묶어서, 내 노트북이든 남의 노트북이든 어디서나 똑같이 실행된다.

컨테이너에 포함되는 것들로는:

- 런타임 환경 (Python, Node.js, Java 등)
- 시스템 라이브러리
- 애플리케이션 코드
- 의존성 패키지
- 환경 설정

이 있다.

> "어? 제 컴퓨터에서는 잘 되는데요?"

협업을 해본 개발자라면 한번쯤은 본인 컴퓨터에서는 정상적으로 작동하지만 남의 컴퓨터에서는 제대로 작동하지않아 이 말을 해본적이 있을것이다. 그리고 그 원인은 다양하지만 일반적으로는 환경이 달라서 이러한 문제가 발생한다.

예를들어,

- Python 3.8에서 작성했는데 서버는 3.10
- 로컬에는 설치된 라이브러리가 서버에는 없음
- Windows에서 개발하고 Linux 서버에 배포

하지만 Docker는 이 모든 환경을 컨테이너에 담아버려서 **어디서든 동일한 실행을 보장**한다.

요약하자면, Docker는 "환경 차이로 인한 문제"를 해결하는 도구이며, 이것이 Docker가 현대 개발 환경에서 필수가 된 이유다. 그렇다면 Docker는 내부적으로 어떻게 작동하는것일까?

## 2. Docker는 어떻게 작동하는가

Docker의 컨테이너 기술은 사실 완전히 새로운 것이 아니다. Linux 커널에 이미 존재하던 여러 기능들을 영리하게 조합한 것이다. 

**핵심 질문**: Docker는 어떻게 하나의 호스트 OS에서 여러 컨테이너를 독립적으로 실행시킬 수 있을까? 각 컨테이너는 마치 자신만의 컴퓨터를 가진 것처럼 동작하는데, 실제로는 모두 같은 Linux 커널을 공유하고 있다.

이 마법의 비결은 세 가지 핵심 기술에 있다.

### Namespaces: 격리의 기술

네임스페이스는 **"같은 컴퓨터를 쓰지만, 서로를 볼 수 없게 만드는"** 기술이다.

#### 구체적인 예시로 이해하기

내 컴퓨터에서 두 개의 컨테이너를 실행한다고 생각해보자:
- 컨테이너 A: Nginx 웹 서버 (80번 포트 사용)
- 컨테이너 B: Node.js 앱 (80번 포트 사용)

일반적으로 하나의 컴퓨터에서 80번 포트를 두 번 사용할 수 없다. 하지만 Docker에서는 가능하다. **Network Namespace** 덕분이다.

각 컨테이너는 자신만의 네트워크 환경을 가진다:
```
호스트 컴퓨터
├── 컨테이너 A의 Network Namespace
│   └── 80번 포트로 Nginx 실행 (컨테이너 A 입장에서는 자기만의 80번 포트)
└── 컨테이너 B의 Network Namespace
    └── 80번 포트로 Node.js 실행 (컨테이너 B 입장에서는 자기만의 80번 포트)
```

컨테이너 A는 자신의 네트워크 환경만 보기 때문에, 컨테이너 B가 80번 포트를 쓰는지 전혀 모른다.

#### 각 Namespace가 Docker에서 하는 역할

**1. PID Namespace - 프로세스 격리**
```bash
# 컨테이너 A에서 프로세스 목록 확인
$ docker exec container-a ps aux
PID   COMMAND
1     nginx
2     worker-process

# 컨테이너 B에서 프로세스 목록 확인
$ docker exec container-b ps aux
PID   COMMAND
1     node app.js
2     child-process
```

두 컨테이너 모두 자신의 프로세스가 PID 1번부터 시작한다고 생각한다. 실제로는 호스트 OS에서 수천, 수만 번째 프로세스일 수 있지만, **각 컨테이너는 자신만의 PID 공간을 가진다**. 그래서 서로의 프로세스를 볼 수 없고, 종료시킬 수도 없다.

**2. Mount Namespace - 파일 시스템 격리**

```bash
컨테이너 A의 파일 시스템:
/
├── app/
│   └── index.html
├── usr/
└── var/

컨테이너 B의 파일 시스템:
/
├── app/
│   └── server.js
├── usr/
└── var/
```

각 컨테이너는 자신만의 `/app` 디렉토리를 가진다. 컨테이너 A에서 파일을 수정해도 컨테이너 B의 파일에는 영향이 없다. **마치 각자 독립된 컴퓨터를 사용하는 것처럼** 동작한다.

**3. Network Namespace - 네트워크 격리**

이미 위에서 설명했듯이, 각 컨테이너는 자신만의 네트워크 인터페이스와 IP 주소를 가진다:

```
호스트: 192.168.1.100
├── 컨테이너 A: 172.17.0.2 (자신만의 IP)
└── 컨테이너 B: 172.17.0.3 (자신만의 IP)
```

**Docker는 이 모든 Namespace를 조합해서 각 컨테이너를 완벽하게 격리시킨다.** 같은 컴퓨터를 쓰지만, 각 컨테이너는 독립된 환경에서 실행되는 것처럼 보인다.

### Control Groups (cgroups): 자원 관리

Namespace가 "격리"를 담당한다면, cgroups는 **"공평한 자원 분배"**를 담당한다.

#### 문제 상황 예시

컨테이너 A가 CPU를 너무 많이 사용하면 어떻게 될까?

```
상황: 컨테이너 A가 무한 루프에 빠짐
결과 (cgroups 없이): 컨테이너 A가 CPU 100%를 독차지
      → 컨테이너 B, C가 멈춤
      → 호스트 시스템도 느려짐
```

이런 상황을 막기 위해 Docker는 cgroups를 사용한다:

```bash
docker run --cpus="1.0" --memory="512m" container
```

이제 컨테이너 A가 아무리 많은 작업을 해도:
- CPU는 1개 코어 이상 사용 못함
- 메모리는 512MB 이상 사용 못함
- 다른 컨테이너들은 정상적으로 작동함

**Docker가 cgroups를 사용하는 이유**: 여러 컨테이너가 하나의 호스트를 공유하면서도, 한 컨테이너가 전체 시스템을 장악하지 못하도록 막기 위해서다.

### Union File System: 효율적인 이미지 관리

Union File System은 **"Docker가 빠르고 디스크 공간을 적게 쓰는 비결"**이다.

#### 문제: 일반적인 방식의 비효율성

만약 Docker가 레이어 개념 없이 작동한다면:

```
Python 앱 A: Ubuntu(2GB) + Python(300MB) + Flask(50MB) + 앱 코드(10MB) = 2.36GB
Python 앱 B: Ubuntu(2GB) + Python(300MB) + Django(80MB) + 앱 코드(15MB) = 2.39GB
Python 앱 C: Ubuntu(2GB) + Python(300MB) + FastAPI(40MB) + 앱 코드(8MB) = 2.35GB

총 디스크 사용량: 7.1GB
```

하지만 세 앱 모두 같은 Ubuntu와 Python을 사용한다. 이걸 세 번 저장하는 건 낭비다.

#### 해결: 레이어 구조

Docker는 이미지를 레이어로 쌓아서 재사용한다:

```
앱 A                앱 B                앱 C
[앱 코드 10MB]     [앱 코드 15MB]     [앱 코드 8MB]
[Flask 50MB]       [Django 80MB]      [FastAPI 40MB]
          ↓              ↓                ↓
        [Python 300MB] ← 공유! (한 번만 저장)
                ↓
        [Ubuntu 2GB] ← 공유! (한 번만 저장)

실제 디스크 사용량: 2GB(Ubuntu) + 300MB(Python) + 50MB + 80MB + 40MB + 33MB = 약 2.5GB
```

**4.6GB 절약!** 이것이 Docker가 가벼운 이유다.

#### 실제 작동 방식

Dockerfile을 작성할 때 각 명령어마다 레이어가 생성된다:

```dockerfile
FROM ubuntu:20.04          # 레이어 1: Ubuntu 베이스
RUN apt-get update         # 레이어 2: 패키지 목록 업데이트
RUN apt-get install python # 레이어 3: Python 설치
COPY app.py /app/          # 레이어 4: 앱 코드 복사
CMD ["python", "app.py"]   # 레이어 5: 실행 명령
```

다음에 `app.py`만 수정해서 다시 빌드하면:
- 레이어 1~3은 캐시에서 재사용 (빠름!)
- 레이어 4~5만 새로 생성
- **빌드 시간이 몇 분에서 몇 초로 단축**

### Docker Architecture: 전체 구조

이제 이 모든 기술이 어떻게 함께 동작하는지 보자:

```
[Docker CLI] 
    ↓ (명령 전달)
[Docker Daemon]
    ↓ (컨테이너 생성 요청)
[containerd]
    ↓ (실제 실행 요청)
[runc]
    ↓ (Namespace + cgroups 설정)
[격리된 컨테이너 프로세스 실행]
```

### 실행 과정: `docker run nginx` 명령 분석

```bash
$ docker run -d -p 8080:80 nginx
```

내부에서 일어나는 일:

1. **Docker CLI → Daemon**: "nginx 컨테이너 실행해줘"
2. **Daemon**: "로컬에 nginx 이미지 있나? → 없으면 Docker Hub에서 다운로드"
3. **Union FS**: nginx 이미지의 모든 레이어를 합쳐서 하나의 파일시스템 생성
4. **containerd**: 컨테이너 생성 준비
5. **runc**: 
   - PID Namespace 생성 → nginx는 자신이 PID 1이라고 생각
   - Network Namespace 생성 → 80번 포트는 이 컨테이너만의 것
   - Mount Namespace 생성 → 독립된 파일시스템
   - cgroups 설정 → CPU/메모리 제한
6. **컨테이너 실행**: 격리된 환경에서 nginx 프로세스 시작
7. **포트 매핑**: 호스트의 8080 포트를 컨테이너의 80 포트에 연결

결과: 외부에서 `localhost:8080`으로 접속하면 컨테이너 내부의 nginx 80번 포트로 연결된다.

### 핵심 정리

Docker 컨테이너는 가상 머신이 아니다. **Linux 커널의 기능(Namespace, cgroups, Union FS)을 활용해 격리된 환경에서 실행되는 일반 프로세스**다.

- **Namespace**: 컨테이너끼리 서로 안 보이게 격리
- **cgroups**: 각 컨테이너가 사용할 자원 제한
- **Union FS**: 이미지를 효율적으로 저장하고 빠르게 실행

이것이 Docker가 가볍고, 빠르고, 효율적인 이유다.