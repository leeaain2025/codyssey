# 1. 프로젝트 개요
터미널, Docker, GitHub의 기본 사용법을 익히고 그 과정을 기록한다.

## 1. 실행환경
- Mac
- zsh
- Docker: 29.4.0
- git: 2.50.1

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


## 4. 트러블슈팅
### 문제 1
  : docker run 중에 port가 이미 점유중이라는 메시지
```bash
docker run -d -p 8080:8000 --name api myserver
5d7eab0a3da228f2a440fe3505a42506b0acdaf7f201b3e2058d2519d5bcde00

What's next:
    Debug this container error with Gordon → docker ai "help me fix this container error"
docker: Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint api (9b0948bd9052b15b02cb4fdecbcd1a1da108818be8ff6f7668d9a6c2bb5d5af8): Bind for 0.0.0.0:8080 failed: port is already allocated
```

- 원인 가설  
  : 앞서 실습에서 docker run을 실행해서 이미 port가 열려 있었던 것이 원인일 것이다.
- 확인
```bash
docker ps -a
```
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED              STATUS        PORTS                                     NAMES
5d7eab0a3da2   myserver   "uvicorn main:app --…"   About a minute ago   Created                                                 api
b63364d17259   nginx      "/docker-entrypoint.…"   14 hours ago         Up 14 hours   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   web
```
역시 web이라는 컨테이너가 14시간 전에 run되어 8080 port를 점유중이다.
- 해결/대안  
해당 docker process를 stop시켰다. 이후 문제없이 docker run을 계속 진행할 수 있었다.
```bash
docker stop web

docker ps -a
```
##### <결과> 
PORTS가 비어있는 것을 볼 수 있다.
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS                      PORTS     NAMES
5d7eab0a3da2   myserver   "uvicorn main:app --…"   3 minutes ago   Created                               api
```
---
### 문제2  
  : 위 문제를 해결하고 다시 docker run을 시도하는데 해당 이름의 컨테이너가 이미 만들어져있다는 에러 발생
- 원인/가설  
  가설: 앞에서 실행했던 docker run이 port 문제로 컨테이너는 생성되었지만 실행은 안된 상태일 것이다.  
        만들어져있다면 실행만 다시 하면 된다.
  추가로 알아본 것: 컨테이너 설정과 파일 시스템은 만들어졌지만 uvicorn 프로세스는 실행되지 않았다.
- 확인
docker에 존재하는 모든 컨테이너 확인(실행 여부 관계없음)
```bash
docker ps -a
```
##### <결과>
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS                     PORTS     NAMES
5d7eab0a3da2   myserver   "uvicorn main:app --…"   49 minutes ago   Created                              api
```

docker가 실행중인 컨테이너만 확인
```bash
docker ps
```
##### <결과>
```bash
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
해당 컨테이너가 실행중이 아님을 확인할 수 있다.

재실행 시도
```bash
docker start api

docker ps -a
```
##### <결과>
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS                      PORTS     NAMES
5d7eab0a3da2   myserver   "uvicorn main:app --…"   52 minutes ago   Exited (3) 5 seconds ago              api
```
(3)번 종료 코드로 종료된 것을 알 수 있다.  종료 코드는 프로세스가 운영체제에 반환하는 숫자로 프로그램마다 의미가 다르기 때문에 여기서는 이 정보로 확인하기보다 docker log로 원인을 파악해보는 것이 나을 것 같다고 판단.
docker의 로그를 확인해본다.
```bash
docker logs api
```
##### <결과>
```bash
ERROR:    Error loading ASGI app. Could not import module "main".
```
- 해결/대안
main.py 파일을 하위 폴더 안에 넣어두어서 docker가 찾지 못해서 발생한 문제였다. 
`COPY . .` 를 실행해도 main.py파일이 없어서 복사가 안되었고 uvicorn이 실행되는 위치에서 main.py를 찾지 못한 것.  
main.py 파일을 프로젝트 루트 폴더로 옮겨오고 이미지를 다시 빌드했다. 그리고 기존 컨테이너는 삭제하고 다시 실행.
```bash
docker build -t myserver .
docker rm -f api
docker run -d -p 8080:8000 --name api myserver
docker ps
```
##### <결과>
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                                         NAMES
baea1e5578e6   myserver   "uvicorn main:app --…"   7 seconds ago   Up 6 seconds   0.0.0.0:8080->8000/tcp, [::]:8080->8000/tcp   api
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
mkdir new_folder
```
##### <결과>
```bash
total 8
drwxr-xr-x@ 4 leeaain  staff   128B Jul 28 21:51 .
drwxr-xr-x@ 3 leeaain  staff    96B Jul 28 12:45 ..
-rw-r--r--@ 1 leeaain  staff   229B Jul 28 13:13 Dockerfile
drwxr-xr-x@ 2 leeaain  staff    64B Jul 28 21:51 new_folder
```

### 이동
```bash
cd new_folder
```
##### <결과>
```bash
total 0
drwxr-xr-x@ 3 leeaain  staff    96B Jul 28 21:52 .
drwxr-xr-x@ 3 leeaain  staff    96B Jul 28 12:45 ..
drwxr-xr-x@ 3 leeaain  staff    96B Jul 28 21:52 new_folder
```

### 복사
```bash
cd new_folder

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
권한 변경 전
```bash
ls -al

drwxr-xr-x@  4 leeaain  staff   128B Jul 29 01:34 .
drwxr-xr-x@  3 leeaain  staff    96B Jul 28 12:45 ..
-rw-r--r--@  1 leeaain  staff     0B Jul 28 22:15 empty.md
```

권한 변경 실행
```bash
chmod 777 empty.md
```
##### <결과>
```bash
ls -al

drwxr-xr-x@  4 leeaain  staff   128B Jul 29 01:34 .
drwxr-xr-x@  3 leeaain  staff    96B Jul 28 12:45 ..
-rwxrwxrwx@  1 leeaain  staff     0B Jul 28 22:15 empty.md
```

# 4. Docker 설치 및 기본 점
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

## 5. Docker 기본 운영 명령 수행
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

```bash
docker stats
```
##### <결과>
```bash
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT    MEM %     NET I/O           BLOCK I/O         PIDS
b63364d17259   web       0.00%     7.73MiB / 7.653GiB   0.10%     13.1kB / 5.18kB   81.9kB / 12.3kB   9
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


## 7. 기존 Dockerfile 기반 커스텀 이미지 제작

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
docker run -d --name alpine -p 8081:80 alp
docker ps -a
```
<결과>
```bash
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS          PORTS                                         NAMES
f032f79ef4bb   alp        "/docker-entrypoint.…"   3 seconds ago    Up 3 seconds    0.0.0.0:8081->80/tcp, [::]:8081->80/tcp       alpine
```

### 어떤 기존 베이스를 선택했는지
nginx:alpine 베이스 선택.  
Alpine Linux 위에 NGINX 웹 서버가 설치된 Docker 이미지.


### 적용한 커스텀 포인트 목적
- index.html 커스텀  
: NGINX 기본 페이지 대신 표



## 8. 포트 매핑 및 접속 증거
![스크린샷]screenshot_00.png



## 9. Docker 볼륨 영속성 검증
### 1. 볼륨 생성
```bash
docker volume create nginx-data
```
<결과>
```bash
docker volume ls

DRIVER    VOLUME NAME
local     nginx-data
```


### 2. 컨테이너 연결
기존에 만든 컨테이너와 생성한 볼륨을 연결해서 실행.
```bash
docker run -d --name alpine -p 8081:80 -v nginx-data:/usr/share/nginx/html alp
```


### 3. 검증
컨테이너 내부에서 index.html 변경
```bash
docker exec -it alpine /bin/sh

cd /usr/share/nginx/html
vi index.html
```
![스크린샷]screenshot_04.png

컨테이너 삭제 전에 확인  
![스크린샷]screenshot_05.png

기존 컨테이너 삭제 후 동일 볼륨으로 새 컨테이너 생성.
```bash
docker rm -f alpine

docker run -d --name alpine -p 8082:80 -v nginx-data:/usr/share/nginx/html alp
```

새 컨테이너에서도 앞에서의 볼륨에 반영한 내용 동일하게 확인되는지 체크.  
![스크린샷]screenshot_06.png


## 10. Git 설정 및 GitHub 연동
- Git 사용자 정보/기본 브랜치 설정
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

- 빌드 및 컨테이너 생성과 실행  
9000포트로 환경변수를 덮어써서 실행
```bash
docker build -t fastapi-uv .
docker run --rm -p 9000:9000 fastapi-uv
```
포트 확인
```bash
docker ps

CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS                    PORTS                                         NAMES
5bbe97f0644b   fastapi-uv           "sh -c 'exec uvicorn…"   4 seconds ago    Up 3 seconds              0.0.0.0:9000->9000/tcp, [::]:9000->9000/tcp   pensive_kowalevski
```



## 5. GitHub SSH 키 설정
설정완료.
![스크린샷]screenshot_08.png













# 웹 서버 컨테이너 만들어보기
웹 서버 컨테이너
### 웹 서버 소스코드 (app/)
```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello from FastAPI!"}
```

### Dockerfile
```Docker
FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:0.7.13 /uv /bin/uv

ENV PYTHONUNBUFFERED=1 \
    PATH="/app/.venv/bin:$PATH"

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-dev

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 빌드/실행 명령 및 결과
```bash
docker build -t myserver .
```
<결과>
```bash
[+] Building 1.5s (15/15) FINISHED                                                                                      docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                                                    0.0s
 => => transferring dockerfile: 385B                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/python:3.12-slim                                                                     1.4s
 => [internal] load metadata for ghcr.io/astral-sh/uv:0.7.13                                                                            1.1s
 => [auth] library/python:pull token for registry-1.docker.io                                                                           0.0s
 => [internal] load .dockerignore                                                                                                       0.0s
 => => transferring context: 75B                                                                                                        0.0s
 => FROM ghcr.io/astral-sh/uv:0.7.13@sha256:6c1e19020ec221986a210027040044a5df8de762eb36d5240e382bc41d7a9043                            0.0s
 => => resolve ghcr.io/astral-sh/uv:0.7.13@sha256:6c1e19020ec221986a210027040044a5df8de762eb36d5240e382bc41d7a9043                      0.0s
 => [stage-0 1/7] FROM docker.io/library/python:3.12-slim@sha256:57cd7c3a7a273101a6485ba99423ee568157882804b1124b4dd04266317710de       0.0s
 => => resolve docker.io/library/python:3.12-slim@sha256:57cd7c3a7a273101a6485ba99423ee568157882804b1124b4dd04266317710de               0.0s
 => [internal] load build context                                                                                                       0.0s
 => => transferring context: 781B                                                                                                       0.0s
 => CACHED [stage-0 2/7] COPY --from=ghcr.io/astral-sh/uv:0.7.13 /uv /bin/uv                                                            0.0s
 => CACHED [stage-0 3/7] WORKDIR /app                                                                                                   0.0s
 => CACHED [stage-0 4/7] COPY pyproject.toml uv.lock ./                                                                                 0.0s
 => CACHED [stage-0 5/7] RUN uv sync --locked --no-install-project                                                                      0.0s
 => CACHED [stage-0 6/7] COPY . .                                                                                                       0.0s
 => CACHED [stage-0 7/7] RUN uv sync --locked                                                                                           0.0s
 => exporting to image                                                                                                                  0.1s
 => => exporting layers                                                                                                                 0.0s
 => => exporting manifest sha256:6d8ab8e25f2fc7cc25eade6fb519de52d1be6ced9ccddf62fb13192768b04e6a                                       0.0s
 => => exporting config sha256:00e2ed44097e50ee1dc5c25b25783a91f394a78b5a04c76d413c44379dbaa582                                         0.0s
 => => exporting attestation manifest sha256:c2eaa896011ffff9e34c0cfb8530072cdf88465bad04510cfe3c9ea3c39085b4                           0.0s
 => => exporting manifest list sha256:bbd2ecc35416517027a925ae6535c2aff271c6f82dee4aa1136617ceb316c145                                  0.0s
 => => naming to docker.io/library/myserver:latest                                                                                      0.0s
 => => unpacking to docker.io/library/myserver:latest   
```

```bash
docker run -d -p 8080:8000 --name api myserver
```
<결과>
```bash
baea1e5578e6ce6436a02fde4a14ea63a420babb3fea4f2fdfa78d2424406225
```

### 포트 매핑 접속 증거
![스크린샷]./screenshot.png

### 바인드 마운트 반영 + 볼륨 영속성 증거
- 바인드 마운트  
message.txt 파일을 하나 생성하고 컨테이너 내부에서 파일내용을 수정해봄.

바인드 마운트 연결
```bash
docker run --rm -p 8080:8000 -v "./message.txt:/app/message.txt" -w /app myserver
```
컨테이너 내부에서 파일 내용 수정
```bash
docker exec -it api /bin/bash
```
```bash
vi message.txt
```

< 호스트 변경 전/후 비교>
![스크린샷]./screenshot_02.png

