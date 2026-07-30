# 1. 프로젝트 개요
터미널, Docker, GitHub의 기본 사용법을 익히고 그 과정을 기록한다.

## 1. 실행환경
- Mac
- zsh
- Docker: 29.4.0
- git: 2.50.1
- 사용 포트: 8000, 8080, 8081 등
  
## 2. 수행 항목 체크리스트
[x] 터미널
[x] 권한
[x] Docker
[x] Dockerfile
[x] port
[x] volume
[x] git
[x] github

## 3. 검증 방법, 결과 위치 링크
### 3.1 디렉토리 구조와 역할
```text
week_1/
├── README.md              # 과제 수행 전체 기록 (터미널/권한/Docker/Compose/Git 로그, 트러블슈팅)
├── screenshot_*.png       # 증거 스크린샷
│
├── try1/                  # 메인 과제
│
└── try2/                   # 보너스 과제 (FastAPI + Compose 실습)
```






## 4. 트러블슈팅
### 문제 1: 8080 포트 충돌
#### 재현 체크리스트
1. try1 폴더에서 시작.
2. 이미지 빌드
```bash
docker build -t myserver .
```
3. 호스트의 8080 포트를 컨테이너의 80포트에 연결
```bash
docker run -d --name api -p 8080:80 myserver
```
4. 포트 충돌 재현을 위해 다른 컨테이너도 동일한 호스트 포트 8080에 연결
```bash
docker run -d --name web -p 8080:80 nginx
```
#### 증상
  : docker run 중에 port가 이미 점유중이라는 메시지
```bash
What's next:
    Debug this container error with Gordon → docker ai "help me fix this container error"
docker: Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint api (9b0948bd9052b15b02cb4fdecbcd1a1da108818be8ff6f7668d9a6c2bb5d5af8): Bind for 0.0.0.0:8080 failed: port is already allocated
```

#### 원인 가설  
  : 앞서 실습에서 docker run을 실행해서 이미 port가 열려 있었을 것.

#### 확인
docker에 존재하는 모든 컨테이너 확인(실행 여부 관계없음)
```bash
docker ps -a
```
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED              STATUS        PORTS                                     NAMES
5d7eab0a3da2   myserver   "/docker-entrypoint.…"   About a minute ago   Created                                                 api
b63364d17259   nginx      "/docker-entrypoint.…"   14 hours ago         Up 14 hours   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   web
```
-> 역시 web이라는 컨테이너가 14시간 전에 run되어 8080 port를 점유중이다.

#### 해결/대안  
해당 docker process를 stop시켰다. 그럼 PORT번호를 물고 있던게 해제된다.
```bash
docker stop web
docker ps -a
```
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS                      PORTS     NAMES
5d7eab0a3da2   myserver   "/docker-entrypoint.…"   2 minutes ago    Created                               api
b63364d17259   nginx      "/docker-entrypoint.…"   14 hours ago     Exited (0) 33 seconds ago             web
```


---
### 문제2  
  : 위 문제를 해결하고 다시 docker run을 시도하는데 해당 이름의 컨테이너가 이미 만들어져있다는 에러 발생
#### 증상
```bash
What's next:
    Debug this container error with Gordon → docker ai "help me fix this container error"
docker: Error response from daemon: Conflict. The container name "/api" is already in use by container "0f5c3538f51a2f888edf91f35ea0439899a81356e21a2ba64627386e461218a1". You have to remove (or rename) that container to be able to reuse that name.
```

#### 원인/가설  
가설: 앞에서 실행했던 docker run이 port 문제로 컨테이너는 생성되었지만 실행은 안된 상태일 것이다.  
	만들어져있다면 실행만 다시 하면 된다.
  	추가로 알아본 것: 컨테이너 설정과 파일 시스템은 만들어졌지만 uvicorn 프로세스는 실행되지 않았다.
	
#### 확인
```bash
docker ps -a
```
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS                      PORTS     NAMES
5d7eab0a3da2   myserver   "/docker-entrypoint.…"   2 minutes ago    Created                               api
b63364d17259   nginx      "/docker-entrypoint.…"   14 hours ago     Exited (0) 33 seconds ago             web
```

docker가 실행중인 컨테이너만 확인
```bash
docker ps
```
실행중인 컨테이너가 없다.

#### 해결/대안
재실행 시도
```bash
docker start api
```

##### <결과>
```bash
docker ps -a
```
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS          PORTS                                     NAMES
5d7eab0a3da2   myserver   "uvicorn main:app --…"   52 minutes ago   Up 3 seconds    0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   api
```

---
### 문제3

#### 재현 체크리스트
1. try2 폴더로 진입
2. docker 이미지 빌드
```bash
docker build -t fastapi-uv .
```
3. 컨테이너 생성과 실행
```bash
docker run -d --name app -p 8080:80 fastapi-uv
```
아무것도 뜨지 않아야 함.

#### 증상
docker 이미지를 빌드 후 컨테이너 생성 & 실행했으나 컨테이너가 돌아가지 않는 상황.

```bash
docker ps
```
아무것도 뜨지 않음.

```bash
docker ps -a
```
```bash
CONTAINER ID   IMAGE        COMMAND                  CREATED              STATUS                          PORTS     NAMES
86407a382df9   fastapi-uv   "sh -c 'exec uvicorn…"   About a minute ago   Exited (3) About a minute ago             app
```

#### 원인/가설
(3)번 종료 코드로 종료된 것을 알 수 있다.  종료 코드는 프로세스가 운영체제에 반환하는 숫자로 프로그램마다 의미가 다르기 때문에 여기서는 이 정보로 확인하기보다 docker log로 원인을 파악해보는 것이 나을 것 같다고 판단.
docker의 로그를 확인해본다.
```bash
docker logs app
```
```bash
ERROR:    Error loading ASGI app. Could not import module "main".
```

#### 해결/대안
main.py 파일을 하위 폴더 안에 넣어두어서 docker가 찾지 못해서 발생한 문제였다. 
`COPY . .` 를 실행해도 main.py파일이 없어서 복사가 안되었고 uvicorn이 실행되는 위치에서 main.py를 찾지 못한 것.  
main.py 파일을 프로젝트 루트 폴더로 옮겨오고 이미지를 다시 빌드했다. 그리고 기존 컨테이너는 삭제하고 다시 실행.
```bash
docker build -t fastapi-uv .
docker rm -f app
docker run -d -p 8080:8000 --name app fastapi-uv
docker ps
```
##### <결과>
```bash
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                                         NAMES
0fb05498c2d1   fastapi-uv   "sh -c 'exec uvicorn…"   16 seconds ago   Up 15 seconds   0.0.0.0:8080->8000/tcp, [::]:8080->8000/tcp   app
```
PORTS도 잘 매핑된 채 서비스가 잘 시작된 것을 볼 수 있다.



# 2. 터미널 조작 로그
### 현재 위치 확인
```bash안
pwd
```
##### <결과>  
```bash
/Users/leeaain/codyssey/week_1
```

### 목록 확인(숨김 파일 포함)
```bash
ls -al
```
##### <결과>
```bash
total 8
drwxr-xr-x@ 3 leeaain  staff   96 Jul 28 21:50 .
drwxr-xr-x@ 3 leeaain  staff   96 Jul 28 12:45 ..
-rw-r--r--@ 1 leeaain  staff  229 Jul 28 13:13 Dockerfile
```

### 생성
```bash
mkdir try1
```
##### <결과>
```bash
total 8
drwxr-xr-x@ 4 leeaain  staff   128B Jul 28 21:51 .
drwxr-xr-x@ 3 leeaain  staff    96B Jul 28 12:45 ..
-rw-r--r--@ 1 leeaain  staff   229B Jul 28 13:13 Dockerfile
drwxr-xr-x@ 2 leeaain  staff    64B Jul 28 21:51 try1
```

### 이동
```bash
cd try1
```
##### <결과>
```bash
total 0
drwxr-xr-x@ 3 leeaain  staff    96B Jul 28 21:52 .
drwxr-xr-x@ 3 leeaain  staff    96B Jul 28 12:45 ..
drwxr-xr-x@ 3 leeaain  staff    96B Jul 28 21:52 try1
```

### 복사
```bash
cd try1

cp Dockerfile Dockerfile.backup
```
##### <결과>
```bash
total 16
drwxr-xr-x@ 4 leeaain  staff   128B Jul 28 22:13 .
drwxr-xr-x@ 4 leeaain  staff   128B Jul 28 21:53 ..
-rw-r--r--@ 1 leeaain  staff   229B Jul 28 13:13 Dockerfile
-rw-r--r--@ 1 leeaain  staff   229B Jul 28 22:13 Dockerfile.backup
```

### 이동/이름변경
```bash
mv Dockerfile.backup test.md
```
##### <결과>
```bash
total 16
drwxr-xr-x@ 4 leeaain  staff   128B Jul 28 22:14 .
drwxr-xr-x@ 4 leeaain  staff   128B Jul 28 21:53 ..
-rw-r--r--@ 1 leeaain  staff   229B Jul 28 13:13 Dockerfile
-rw-r--r--@ 1 leeaain  staff   229B Jul 28 22:13 test.md
```

### 삭제
```bash
rm test.md
```
##### <결과>
```bash
total 8
drwxr-xr-x@ 3 leeaain  staff    96B Jul 28 22:15 .
drwxr-xr-x@ 4 leeaain  staff   128B Jul 28 21:53 ..
-rw-r--r--@ 1 leeaain  staff   229B Jul 28 13:13 Dockerfile
```

### 빈 파일 생성
```bash
touch empty.md
```
##### <결과>
```bash
total 8
drwxr-xr-x@ 4 leeaain  staff   128B Jul 28 22:15 .
drwxr-xr-x@ 4 leeaain  staff   128B Jul 28 21:53 ..
-rw-r--r--@ 1 leeaain  staff   229B Jul 28 13:13 Dockerfile
-rw-r--r--@ 1 leeaain  staff     0B Jul 28 22:15 empty.md
```

### 파일 내용 확인
```bash
cat empty.md
```
##### <결과>
```bash

```


# 3. 권한 실습 및 증거 기록
### 이론
`rwx` 비트 
- `r`: 읽기. 숫자 4. 파일 내용 읽기 가능한 권한. 파일 이름과 목록 조회도 해당.
- `w`: 쓰기. 숫자 2. 파일 내용 쓰기 가능한 권한. 파일 생성과 삭제, rename도 여기에 해당.
- `x`: 실행. 숫자 1. 파일을 프로그램으로 실행 가능한 권한. 디렉토리에 진입하고 내부 항목에 접근하는 것도 여기 해당.

권한값을 더해서 한자리 숫자로 표현하며, 소유자/그룹/기타사용자의 3자리로 최종적으로 표현된다.  

<정리표>
| 권한    | 문자 표기       | 의미                         | 일반적인 사용 사례              |
| ----- | ----------- | -------------------------- | ----------------------- |
| `755` | `rwxr-xr-x` | 소유자는 읽기·쓰기·실행, 나머지는 읽기·실행  | 디렉터리, 공개 실행 파일·스크립트     |
| `644` | `rw-r--r--` | 소유자는 읽기·쓰기, 나머지는 읽기        | 일반 문서, 설정 파일, HTML 파일   |
| `700` | `rwx------` | 소유자만 모든 권한 보유              | 개인 디렉터리, 비공개 실행 스크립트    |
| `600` | `rw-------` | 소유자만 읽기·쓰기                 | SSH 개인 키, 비밀번호·토큰 파일    |
| `775` | `rwxrwxr-x` | 소유자와 그룹은 모든 권한, 나머지는 읽기·실행 | 그룹 공동 작업 디렉터리           |
| `664` | `rw-rw-r--` | 소유자와 그룹은 읽기·쓰기, 나머지는 읽기    | 그룹 공동 편집 파일             |
| `777` | `rwxrwxrwx` | 모든 사용자가 읽기·쓰기·실행           | 보안 위험이 커서 일반적으로 사용하지 않음 |

디렉토리를 읽고 이용하려면 `r-x`가 함께 필요함.


### 실습
권한 변경 전
```bash
ls -al

drwxr-xr-x@  4 leeaain  staff   128B Jul 29 01:34 .
drwxr-xr-x@  3 leeaain  staff    96B Jul 28 12:45 ..
-rw-r--r--@  1 leeaain  staff     0B Jul 28 22:15 empty.md
```

권한 변경 실행
```bash
chmod 600 empty.md
```

##### <결과>
```bash
ls -al

-rw-------@ 1 leeaain  staff     0B Jul 29 01:54 README.md
```
위 권한에 대한 설명
- 파일 소유자만 읽고 수정할 수 있으며, 나머지 사용자는 아무 권한도 없음.  
- 실행파일에는 적합하지 않고 소유자 외에 공개하고 싶지 않은 파일에 적합한 설정.





# 4. Docker 설치 및 기본 점검
### Docker 버전 확인
```bash
docker --version
```
##### <결과>
```bash
Docker version 29.4.0, build 9d7ad9f
```

### Docker 데몬 동작 여부 확인
```bash
docker info
```

##### <결과>
```bash
Client:
 Version:    29.4.0
 Context:    desktop-linux
 Debug Mode: false
 Plugins:
  agent: Docker AI Agent Runner (Docker Inc.)
    Version:  v1.42.0
    Path:     /Users/leeaain/.docker/cli-plugins/docker-agent
  ai: Docker AI Agent - Ask Gordon (Docker Inc.)
    Version:  v1.20.2
    Path:     /Users/leeaain/.docker/cli-plugins/docker-ai
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.33.0-desktop.1
    Path:     /Users/leeaain/.docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v5.1.1
    Path:     /Users/leeaain/.docker/cli-plugins/docker-compose
  debug: Get a shell into any image or container (Docker Inc.)
    Version:  0.0.47
    Path:     /Users/leeaain/.docker/cli-plugins/docker-debug
  desktop: Docker Desktop commands (Docker Inc.)
    Version:  v0.3.0
    Path:     /Users/leeaain/.docker/cli-plugins/docker-desktop
  dhi: CLI for managing Docker Hardened Images (Docker Inc.)
    Version:  v0.0.2
    Path:     /Users/leeaain/.docker/cli-plugins/docker-dhi
  extension: Manages Docker extensions (Docker Inc.)
    Version:  v0.2.31
    Path:     /Users/leeaain/.docker/cli-plugins/docker-extension
  init: Creates Docker-related starter files for your project (Docker Inc.)
    Version:  v1.4.0
    Path:     /Users/leeaain/.docker/cli-plugins/docker-init
  mcp: Docker MCP Plugin (Docker Inc.)
    Version:  v0.40.3
    Path:     /Users/leeaain/.docker/cli-plugins/docker-mcp
  model: Docker Model Runner (Docker Inc.)
    Version:  v1.1.29
    Path:     /Users/leeaain/.docker/cli-plugins/docker-model
  offload: Docker Offload (Docker Inc.)
    Version:  v0.5.82
    Path:     /Users/leeaain/.docker/cli-plugins/docker-offload
  pass: Docker Pass Secrets Manager Plugin (beta) (Docker Inc.)
    Version:  v0.0.25
    Path:     /Users/leeaain/.docker/cli-plugins/docker-pass
  sandbox: Docker Sandbox (Docker Inc.)
    Version:  v0.12.0
    Path:     /Users/leeaain/.docker/cli-plugins/docker-sandbox
  sbom: View the packaged-based Software Bill Of Materials (SBOM) for an image (Anchore Inc.)
    Version:  0.6.0
    Path:     /Users/leeaain/.docker/cli-plugins/docker-sbom
  scout: Docker Scout (Docker Inc.)
    Version:  v1.20.3
    Path:     /Users/leeaain/.docker/cli-plugins/docker-scout

Server:
 Containers: 1
  Running: 1
  Paused: 0
  Stopped: 0
 Images: 2
 Server Version: 29.4.0
 Storage Driver: overlayfs
  driver-type: io.containerd.snapshotter.v1
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
 CDI spec directories:
  /etc/cdi
  /var/run/cdi
 Discovered Devices:
  cdi: docker.com/gpu=webgpu
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: dea7da592f5d1d2b7755e3a161be07f43fad8f75
 runc version: v1.3.4-0-gd6d73eb8
 init version: de40ad0
 Security Options:
  seccomp
   Profile: builtin
  cgroupns
 Kernel Version: 6.12.76-linuxkit
 Operating System: Docker Desktop
 OSType: linux
 Architecture: aarch64
 CPUs: 8
 Total Memory: 7.653GiB
 Name: docker-desktop
 ID: 2215097e-b8c5-4dd2-b688-4aaf02cdb689
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 HTTP Proxy: http.docker.internal:3128
 HTTPS Proxy: http.docker.internal:3128
 No Proxy: hubproxy.docker.internal
 Labels:
  com.docker.desktop.address=unix:///Users/leeaain/Library/Containers/com.docker.docker/Data/docker-cli.sock
 Experimental: false
 Insecure Registries:
  hubproxy.docker.internal:5555
  ::1/128
  127.0.0.0/8
 Live Restore Enabled: false
 Firewall Backend: iptables
```

# 5. Docker 기본 운영 명령 수행
### 이미지: 다운로드/목록 확인
```bash
docker images
```
##### <결과>
```bash
                                                                                                 i Info →   U  In Use
IMAGE          ID             DISK USAGE   CONTENT SIZE   EXTRA
nginx:latest   5a88c9c45479        259MB         64.3MB    U   
```

### 컨테이너: 실행/중지/목록 확인
```bash
docker ps -a
```
##### <결과>
```bash
CONTAINER ID   IMAGE     COMMAND                  CREATED        STATUS        PORTS                                     NAMES
b63364d17259   nginx     "/docker-entrypoint.…"   14 hours ago   Up 14 hours   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   web
```

### 운영: 로그 확인
```bash
docker logs b63364d17259
```
##### <결과>
```bash
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/07/27 23:38:38 [notice] 1#1: using the "epoll" event method
2026/07/27 23:38:38 [notice] 1#1: nginx/1.31.3
2026/07/27 23:38:38 [notice] 1#1: built by gcc 14.2.0 (Debian 14.2.0-19) 
2026/07/27 23:38:38 [notice] 1#1: OS: Linux 6.12.76-linuxkit
2026/07/27 23:38:38 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2026/07/27 23:38:38 [notice] 1#1: start worker processes
2026/07/27 23:38:38 [notice] 1#1: start worker process 29
2026/07/27 23:38:38 [notice] 1#1: start worker process 30
2026/07/27 23:38:38 [notice] 1#1: start worker process 31
2026/07/27 23:38:38 [notice] 1#1: start worker process 32
2026/07/27 23:38:38 [notice] 1#1: start worker process 33
2026/07/27 23:38:38 [notice] 1#1: start worker process 34
2026/07/27 23:38:38 [notice] 1#1: start worker process 35
2026/07/27 23:38:38 [notice] 1#1: start worker process 36
192.168.65.1 - - [27/Jul/2026:23:56:15 +0000] "GET / HTTP/1.1" 200 896 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/150.0.0.0 Safari/537.36" "-"
2026/07/27 23:56:15 [error] 29#29: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.65.1, server: localhost, request: "GET /favicon.ico HTTP/1.1", host: "localhost:8080", referrer: "http://localhost:8080/"
192.168.65.1 - - [27/Jul/2026:23:56:15 +0000] "GET /favicon.ico HTTP/1.1" 404 555 "http://localhost:8080/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/150.0.0.0 Safari/537.36" "-"
192.168.65.1 - - [27/Jul/2026:23:56:30 +0000] "\x16\x03\x01\x01:\x01\x00\x016\x03\x03~\xCE(\xFEK!\xE3\xD4\xE1\x03\xB5\x8B\xE7Y\xA2\x17\xA7\x185$>@\x11J\xE3\xD3]\xB0\xAC,\x96\x9C " 400 157 "-" "-" "-"
192.168.65.1 - - [27/Jul/2026:23:59:16 +0000] "GET / HTTP/1.1" 200 896 "-" "curl/8.7.1" "-"
```

### 운영: 리소스 확인
```bash
docker stats
```
##### <결과>
```bash
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT     MEM %     NET I/O         BLOCK I/O    PIDS
7c7d0f6dab9f   api       0.82%     36.98MiB / 7.653GiB   0.47%     1.17kB / 126B   0B / 8.2MB   1
```


# 6. 컨테이너 실행 실습
### hello-world 실행
```bash
docker run --name hello-test hello-world
```
##### <결과>
```bash
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
58dee6a49ef1: Pull complete 
c3bdf82c34d1: Download complete 
Digest: sha256:c3cbe1cc1aa588a64951ac6286e0df7b27fe2e6324b1001c619bb358770c0178
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```


### ubuntu 컨테이너 실행 후 내부에서 간단한 명령 실행
```bash
docker run -dit --name ubuntu-test ubuntu:24.04 /bin/bash
docker exec -it ubuntu-test /bin/bash
```
##### <결과>
```bash
root@c839b3b76ff6:/# pwd
/
root@c839b3b76ff6:/# ls -al
total 56
drwxr-xr-x   1 root root 4096 Jul 28 20:37 .
drwxr-xr-x   1 root root 4096 Jul 28 20:37 ..
-rwxr-xr-x   1 root root    0 Jul 28 20:37 .dockerenv
lrwxrwxrwx   1 root root    7 Apr 22  2024 bin -> usr/bin
drwxr-xr-x   2 root root 4096 Apr 22  2024 boot
drwxr-xr-x   5 root root  360 Jul 28 20:37 dev
drwxr-xr-x   1 root root 4096 Jul 28 20:37 etc
drwxr-xr-x   3 root root 4096 Jun 10 02:16 home
lrwxrwxrwx   1 root root    7 Apr 22  2024 lib -> usr/lib
drwxr-xr-x   2 root root 4096 Jun 10 02:09 media
drwxr-xr-x   2 root root 4096 Jun 10 02:09 mnt
drwxr-xr-x   2 root root 4096 Jun 10 02:09 opt
dr-xr-xr-x 230 root root    0 Jul 28 20:37 proc
drwx------   2 root root 4096 Jun 10 02:16 root
drwxr-xr-x   4 root root 4096 Jun 10 02:16 run
lrwxrwxrwx   1 root root    8 Apr 22  2024 sbin -> usr/sbin
drwxr-xr-x   2 root root 4096 Jun 10 02:09 srv
dr-xr-xr-x  11 root root    0 Jul 28 20:37 sys
drwxrwxrwt   2 root root 4096 Jun 10 02:16 tmp
drwxr-xr-x  11 root root 4096 Jun 10 02:09 usr
drwxr-xr-x  11 root root 4096 Jun 10 02:16 var
```


### 컨테이너 종료와 유지의 차이 설명
- `docker exec`은 새로운 shell을 생성해서 컨테이너에 접속. 
	`exit`를 통해 빠져나오면 해당 shell만 종료되고 기존 메인 프로세스는 유지된다.
- `docker attach`는 새로운 shell을 만들지 않고 컨테이너의 기존 주 프로세스인 bash에 직접 연결한다. 
	`exit`를 통해 빠져나오면 기존 메인 프로세스까지 함께 종료된다.
### 이미지와 컨테이너의 차이
- 이미지: 설계도. 소스코드, 라이브러리, 설정 및 기본 실행 명령을 읽기 전용 레이어로 묶어놓은 불변 템플릿.
- 컨테이너: 이미지를 기반으로 생성되고 실행되는 인스턴스.


# 7. 기존 Dockerfile 기반 커스텀 이미지 제작
1. try1 폴더로 진입
2. 
Dockerfile
```docker
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

index.html
```html
<h1>My Custom NGINX Server</h1>
<p>Docker custom image is running.</p>
```

```bash
docker build -t alpine .
docker run -d --name week1-nginx-container -p 8081:80 myserver
docker ps
```
<결과>
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED              STATUS              PORTS                                     NAMES
bd5235b01cfc   myserver   "/docker-entrypoint.…"   About a minute ago   Up About a minute   0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   week1-nginx-container
```

### 어떤 기존 베이스를 선택했는지
nginx:alpine 베이스 선택.  
Alpine Linux 위에 NGINX 웹 서버가 설치된 Docker 이미지.


### 적용한 커스텀 포인트 목적
- index.html 커스텀  
: NGINX 기본 페이지 대신 표시될 html 내용.



# 8. 포트 매핑 및 접속 증거
```bash
curl http://localhost:8081
```
```bash
<h1>My Custom NGINX Server</h1>
<p>Docker custom image is running.</p>
```
![스크린샷]screenshot_00.png



# 9. Docker 볼륨 영속성 검증
### 0. 바인드 마운트 개념
Docker 볼륨 대신 호스트의 특정 디렉토리를 컨테이너에 직접 연결하는 것.  
자주 수정해야 하는 개발중인 상황에 적합하다.  
Docker가 지속적으로 관리해야 하는 데이터(DB)는 Docker 볼륨이 더 적합하다.

#### 장점  
- 호스트에서 파일을 직접 확인하고 수정하기 쉽다.
- 소스 코드나 설정 파일처럼 자주 변경되는 파일을 연결하기 편하다.
- 일반 디렉토리이므로 별도의 볼륨 복원 과정 없이 파일을 백업할 수 있다.

#### 단점
- 호스트의 디렉토리 구조와 권한 설정에 영향을 받는다.
- 운영체제가 달라지면 경로나 권한 문제가 발생할 수 있다.
- 컨테이너가 호스트 파일에 직접 접근하므로 데이터 변경 혹은 삭제 리스크가 있다.

#### 명령 예시  
```bash
mkdir -p "$(pwd)/nginx-data"

docker run -d \
  --name week1-nginx-container \
  -p 8080:80 \
  -v "$(pwd)/nginx-data:/usr/share/nginx/html" \
  nginx
```


### 실습

<준비>  
기존에 동작중인 컨테이너가 있다면 제거
```bash
docker rm -f week1-nginx-container
```

### 1. 볼륨 생성
```bash
docker volume create nginx-data
docker volume ls
```
```bash
DRIVER    VOLUME NAME
local     nginx-data
```


### 2. 컨테이너 연결
기존에 빌드해둔 이미지로 컨테이너를 생성하면서 볼륨도 연결해서 실행.
```bash
docker run -d --name week1-nginx-container -p 8082:80 -v nginx-data:/usr/share/nginx/html myserver
```


### 3. 컨테이너와 볼륨 연결 검증
#### 1. 컨테이너 내부에서 index.html 변경
```bash
docker exec -it week1-nginx-container /bin/sh
cd /usr/share/nginx/html
vi index.html
```

<변경 전>
```html<h1>My Custom NGINX Server</h1>
<p>Docker custom image is running.</p>
```

<변경 후>
```html
<h1>My Custom NGINX Server</h1>
<p>Docker custom image is running.</p>

<span>Edited in volume!</span>
```

#### 2. 변경됐는지 확인  
```bash
curl http://localhost:8082
```
```html
<h1>My Custom NGINX Server</h1>
<p>Docker custom image is running.</p>

<span>Edited in volume!</span>
```

![스크린샷]screenshot_04.png
![스크린샷]screenshot_05.png

#### 3. 확인
0. 기존 컨테이너 삭제
```bash
docker rm -f week1-nginx-container
```

1. 일반적인 방법으로 새 컨테이너 생성
```bash
docker run -d --name test -p 8000:80 myserver
docker exec -it test /bin/sh
cd usr/share/nginx/html
cat index.html
```
아래처럼 조금전에 변경한 내용이 날아가고 없어야 한다.
```html
<h1>My Custom NGINX Server</h1>
<p>Docker custom image is running.</p>
```

2. 동일 볼륨으로 새 컨테이너 생성.
```bash
docker run -d --name week1-nginx-container -p 8082:80 -v nginx-data:/usr/share/nginx/html myserver
cd usr/share/nginx/html
cat index.html
```
아래처럼 수정한 내용이 보여야 한다.
```html
<h1>My Custom NGINX Server</h1>
<p>Docker custom image is running.</p>

<span>Edited in volume!</span>
```
![스크린샷]screenshot_06.png


### 4. 볼륨 백업하기
먼저 데이터 변경을 막기 위해 연결된 컨테이너를 중지
```bash
docker stop week1-nginx-container
```

볼륨을 백업한다.  
현재 폴더에 nginx-data-backup.tar.gz 파일을 생성.
이 때 임시 Alpine 컨테이너를 이용하여 압축 파일로 백업한다.
```bash
docker run --rm \
  -v nginx-data:/data:ro \
  -v "$(pwd):/backup" \
  alpine \
  tar czf /backup/nginx-data-backup.tar.gz -C /data .
```
Docker volume은 독립적으로 명령어를 실행할 수 있는 프로그램이 아니라 단순한 저장 공간으로,  
파일 목록 조회나 압축등의 기능이 없다.  
그래서 볼륨을 임시 컨테이너에 연결하고 컨테이너 안의 `tar` 명령을 이용한다.  
Alpine은 이미지 크기가 작고 `tar`같은 기본 도구가 포함되어 있어 임시 컨테이너로 사용하기에 적합하다. Ubuntu같은 이미지도 good.


### 5. 볼륨 복원
안전한 복원을 위해 새 볼륨을 생성한다.  
기존 데이터를 건드리지 않고 백업 파일이 제대로 복원되는지 검증하기 위해서다.
기존 볼륨에 바로 복원하면 기존 파일과 백업 파일이 섞이며 같은 이름의 파일이 덮어씌워지는 등 데이터가 망가질 위험이 있다.
```bash
docker volume create nginx-data-restored
```

기존에 백업해둔 내용으로부터 복원한다.
```bash
docker run --rm \
  -v nginx-data-restored:/data \
  -v "$(pwd):/backup:ro" \
  alpine \
  tar xzf /backup/nginx-data-backup.tar.gz -C /data
```

잘 복원됐는지 확인
```bash
docker run --rm \
  -v nginx-data-restored:/data \
  alpine \
  ls -la /data
```
```bash
docker run --rm \
  -v nginx-data-restored:/data \
  alpine \
  cat /data/index.html
```

### 6. 검증을 위해 index.html 파일 변경
중지해뒀던 컨테이너 재시작 후 내부 진입
```bash
docker start week1-nginx-container
docker exec -it week1-nginx-container /bin/sh
cd usr/share/nginx/html
vi index.html
```

<변경 전>
```html
<h1>My Custom NGINX Server</h1>
<p>Docker custom image is running.</p>

<span>Edited in volume!</span>
```
<변경 후>
```html
<h1>My Custom NGINX Server</h1>
```

### 7. 
복원한 볼륨으로 새 컨테이너 실행.  
기존의 컨테이너와 겹치지 않게 컨테이너 이름과 포트번호를 8083으로 수정한다.
```bash
docker run -d --name week1-nginx-restored -p 8083:80 -v nginx-data-restored:/usr/share/nginx/html myserver
```
index.html 확인
```bash
docker exec week1-nginx-restored \
  cat /usr/share/nginx/html/index.html

curl http://localhost:8083
```

복원된 컨테이너에서는 원본 파일을 수정하기 전, 즉 백업 당시의 내용이 출력되어야 한다.
```html
<h1>My Custom NGINX Server</h1>
<p>Docker custom image is running.</p>

<span>Edited in volume!</span>
```




## 10. Git 설정 및 GitHub 연동

git 원격 repo 주소: https://github.com/leeaain2025/codyssey

### Git 사용자 정보/기본 브랜치 설정
```bash
git config --global user.name "leeaain2025"
git config --global user.mail "leeaain2025@gmail.com"
git config --global init.defaultBranch main
```
##### <결과>
```bash
git config --list
```
![스크린샷]screenshot_06.png

### git push 이력
```bash
git log --oneline origin/main
```
##### <결과>
```bash
061c923 (HEAD -> main, origin/main) modified README.md
6a9278d modified README.md
58b0006 modified README.md
de56c11 complete README.md
7260497 submit
6c8b58d Initial commit
```





# 보너스 과제 (선택)
## 1. Docker Compose 기초
try2 폴더에서 실습 완료  
- docker-compose.yml 작성 
```docker
services:
  web:
    image: nginx:alpine
    container_name: nginx-web
    ports:
      - "8081:80"
    volumes:
      - nginx-data:/usr/share/nginx/html

volumes:
  nginx-data:
```
- 실행
```bash
docker compose up -d
```
- 확인
```bash
docker compose ps
```
##### <결과>
```bash
NAME        IMAGE          COMMAND                  SERVICE   CREATED          STATUS          PORTS
nginx-web   nginx:alpine   "/docker-entrypoint.…"   web       12 seconds ago   Up 11 seconds   0.0.0.0:8081->80/tcp, [::]:8081->80/tcp
```



## 2. Docker Compose 멀티 컨테이너
호스트 → FastAPI/Uvicorn → PostgreSQL 구조.  
`compose.yml` 내용

```docker
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      DB_HOST: db
      DB_NAME: appdb
      DB_USER: appuser
      DB_PASSWORD: password
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: password
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  db-data:
```

- 빌드 및 컨테이너 생성과 실행
```bash
docker compose up -d --build
```

- 확인
```bash
docker compose ps
```
##### <결과>
```bash
NAME         IMAGE                COMMAND                  SERVICE   CREATED         STATUS                   PORTS
try2-db-1    postgres:17-alpine   "docker-entrypoint.s…"   db        5 minutes ago   Up 5 minutes (healthy)   5432/tcp
try2-web-1   try2-web             "uvicorn main:app --…"   web       5 minutes ago   Up 5 minutes             0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp
```

- db 컨테이너 확인
```bash
docker compose exec web getent hosts db
```
##### <결과>
```bash
172.20.0.2    db
```
Compose 내부 DNS가 db라는 서비스 이름을 DB 컨테이너 IP로 변환했음을 확인.

- web 컨테이너에서 db 컨테이너와의 통신 확인
```bash
docker compose exec web /bin/sh

root@500cf7acbcd5:/app# getent hosts db
```

##### <결과>
```bash
172.18.0.2      db
```

- web 컨테이너에서 db 컨테이너에 `INSERT`하기
```bash
docker compose exec -it web /bin/bash

PGPASSWORD=password psql -h db -U appuser -d appdb
```
postgreSQL에 INSERT하기
```sql
INSERT INTO messages (content)
VALUES ('Inserted from web container');

SELECT id, content
FROM messages
ORDER BY id;
```

- db 컨테이너에서 결과 확인하기  
db 컨테이너에 접속해서 postgre SQL에 접속 후 조회하기
```bash
docker compose exec db /bin/sh

/ # psql -U appuser -d appdb
SELECT * FROM messages;
```
##### <결과>
```bash
 id |           content           
----+-----------------------------
  1 | Inserted from web container
(1 row)
```
web 컨테이너에서 실행한 DB INSERT 구문이 잘 실행된 결과를 볼 수 있다.


## 3. Compose 운영 명령어 습득
### 1. compose로 컨테이너 생성과 실행
`docker compose up`
compose.yml에 정의된 서비스의 컨테이너를 생성하고 실행.  
`--build` 옵션을 붙이면 이미지 빌드까지 수행한다.

### 2. 현재 프로젝트의 컨테이너 상태 확인 
`docker compose ps`

### 3. 로그 확인
```bash
# 전체 서비스 로그 확인
docker compose logs

# 서비스별 확인
docker compose logs web
docker compose logs db
```
로그가 너무 길면 `--tail=50` 파라미터로 출력 길이를 조절.


### 4. 실행중인 서비스(컨테이너) 중지
```bash
docker compose down
```
빌드한 이미지와 db등의 볼륨, 볼륨 내부의 데이터는 기본적으로 유지된다.  
만약 볼륨과 데이터까지 삭제하려면
```bash
docker compose down -v
```
compose가 빌드한 이미지까지 삭제하려면(공식저장소에서 내려받은 postgres:17-alpine같은 이미지는 유지)
```bash
docker compose down --rmi local
```
모든 이미지 삭제
```bash
docker compose down --rmi all
```
이미지 목록 확인
```bash
docker image ls
```

빌드한 이미지와 볼륨+데이터까지 삭제
```bash
docker compose down -v --rmi local
```




## 4. 환경 변수 활용
Dockerfile
```docker
FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:0.7.13 /uv /bin/uv

ENV PYTHONUNBUFFERED=1 \
    PATH="/app/.venv/bin:$PATH" \
    SERVER_PORT=8000

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-install-project

COPY . .

RUN uv sync --locked

EXPOSE 8000

CMD ["sh", "-c", "exec uvicorn main:app --host 0.0.0.0 --port \"$SERVER_PORT\""]
```
위 명세대로 이미지를 빌드하면 ENV에 기재된 환경변수들이 기본값으로 저장된다.
```bash
docker build -t fastapi-uv .
```

- 빌드 및 컨테이너 생성과 실행  
`docker run`은 컨테이너를 생성하고 실행한다.  

이 때, Dockerfile의 ENV를 docker run에서 덮어쓸 수 있다.
아래 구문은 컨테이너 내부의 `PORT` 환경변수 값을 덮어쓴다.
```bash
docker run --rm -e SERVER_PORT=9000 fastapi-uv
```

아래 구문은 호스트 포트와 컨테이너 포트를 연결해서 실행하는 것으로, 환경변수를 변경하지는 않는다.  

```bash
docker run --rm -p 9000:9000 fastapi-uv
```
-p 옵션은 컨테이너의 네트워크를 외부에 어떻게 노출할지 정하는 것으로,  
호스트:컨테이너포트 순서이다.  
호스트 포트는 내 PC의 포트이고,  
컨테이너 포트란 컨테이너 내부에서 프로그램이 실제로 listen하는 포트이다.  
예를 들어 컨테이너 내부에서 fastapi가 실제 사용하는 포트는 8000인데  
```bash
docker run -p 9000:9000 fastapi-uv
```
이렇게 지정한다면 `-p 9000:9000` 연결은 만들어지지만 응답은 오지 않는다.


포트 확인  
```bash
docker ps

CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS                    PORTS                                         NAMES
5bbe97f0644b   fastapi-uv           "sh -c 'exec uvicorn…"   4 seconds ago    Up 3 seconds              0.0.0.0:9000->9000/tcp, [::]:9000->9000/tcp   pensive_kowalevski
```



## 5. GitHub SSH 키 설정
설정완료.
![스크린샷]screenshot_08.png











# 추가
## 1. 포트 노출 이유
- 포트 노출 이유(네임스페이스·보안 관점)
  : 환경변수로 입력으로 달라지는 포트 번호를 보이기 위함이며 보안상 리스크를 인지하고 있음. 

## 2. 절대 경로와 상대 경로 선택 기준
`-v` 옵션은 다음과 같이 구성된다.
```
호스트 경로:컨테이너 경로
```
- 절대 경로는 /부터 전체 위치를 작성하는 방식이다.
	- $(pwd)는 현재 작업 디렉토리의 절대 경로를 출력한다.
	- `/backup`은 컨테이너 내부의 절대 경로이다.
- 상대 경로는 현재 위치를 기준으로 하므로, 명령 실행 위치에 따라 대상이 달라질 수 있다.

### 경로 선택 기준
- 컨테이너 경로는 `/backup`, `/data`처럼 절대 경로로 작성한다.
- 프로젝트 파일은 $(pwd)를 사용하면 사용자마다 다른 프로젝트 경로를 직접 작성하지 않아도 된다.  
	단, $(pwd)는 실행 위치에 따라 달라지므로 명령을 실행할 기준 위치를 정해야 한다.
- 서버의 고정된 데이터는 `/srv/nginx-data`처럼 명시적인 절대 경로를 사용한다.
- Nginx처럼 정해진 디렉토리의 파일을 읽는 프로그램에 데이터를 연결할 때는, 프로그램이 실제로 사용하는 컨테이너 내부 경로를 확인하여 지정해야 한다.
