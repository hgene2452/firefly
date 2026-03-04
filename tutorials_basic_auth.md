# 4. Tutorials (Basic Auth)

## 1 FireFly 인증

[Basic Auth - Hyperledger FireFly](https://hyperledger.github.io/firefly/latest/tutorials/basic_auth/)

### 1.1 FireFly 인증 필요성

- FF는 기본적으로 HTTP API가 열려 있는 상태로 실행된다
- 따라서 인증을 적용하지 않으면 제공하는 기능(e.g. Token Mint)에 제어 없이 접근 가능해진다
- 이에 Auth를 적용하여 사용자 인증을 통과한 요청만 API를 사용할 수 있도록 구성하는 것이 필요하다

---

### 1.2 Auth 플러그인 구조

- FF는 인증을 플러그인 기반 구조로 제공한다
- Auth 플러그인 특징 :
    - FF Core 내부 플러그인 구조로, 별도의 컨테이너 없이 Core 설정으로 활성화 가능하다
    - FF는 Auth 플러그인 인터페이스를 제공하고, 이를 구현한 인증 방식(e.g. jwt, basic)을 선택하여 사용하도록 설계돼있다

---

### 1.3 Auth 적용 위치

- FF는 2가지 레벨에서 인증을 적용할 수 있다
- HTTP Listener 레벨 :
    - 특정 HTTP Listener(포트)로 들어오는 요청 전체에 인증을 요구한다
    - Main API : FF의 기본 API 인터페이스 (e.g. 토큰풀 관리, tx 생성, 컨트랙트 invoke, 이벤트 조회)
    - SPI(Service Provider Interface) : FF 내부 관리/운영 기능 제공 (e.g. Namespace 관리, 플러그인 설정)
    - Metrics API : FF 모니터링용 API
    - 각 HTTP Listener는 서로 다른 인증 정책을 설정할 수 있다
    
    ```json
    Client
       ↓
    HTTP Listener
       ↓
    Listener Auth
       ↓
    FireFly Core
    ```
    
- Namespace 레벨 :
    - 동일한 FF 노드(동일한 HTTP Listener)에서도 Namespace 단위로 사용자 인증 정책을 적용할 수 있다
    - 왜 필요한가? 하나의 FF 노드를 여러 팀이 공유하는 경우, 서로 다른 사용자 인증 정책을 사용하도록하는 멀티 테넌트 구조를 구성할 수 있다
    
    ```json
    Client
       ↓
    HTTP Listener
       ↓
    Listener Auth
       ↓
    Namespace Routing
       ↓
    Namespace Auth
       ↓
    FireFly Core
    ```
    

## 2 FireFly Basic Auth

### 2.1 FF Auth 플러그인

[https://github.com/hyperledger/firefly-common/blob/main/pkg/auth/plugin.go](https://github.com/hyperledger/firefly-common/blob/main/pkg/auth/plugin.go)

- FF는 인증 로직을 Core 내부에 고정하지 않고 플러그인 기반 구조로 설계돼있다
- FF Core는 Auth 플러그인 인터페이스를 정의하고, 실제 인증 방식은 해당 인터페이스를 구현한 플러그인(e.g. jwt, basic)을 통해 동작한다
- Auth 플러그인이 구현해야 하는 함수 :
    - Name() : 플러그인 이름 반환
    - InitConfig(config) : 플러그인이 사용하는 설정 항목을 등록한다
    - Init(context, name, config) : FF 시작 시 호출되는 플러그인 초기화 단계
    - Authorize(context, request) : FF API 요청이 들어올 때마다 호출되는 인증 함수

---

### 2.2 Basic Auth 구현 로직

[https://github.com/hyperledger/firefly-common/blob/main/pkg/auth/basic/basic_auth.go](https://github.com/hyperledger/firefly-common/blob/main/pkg/auth/basic/basic_auth.go)

- FF는 Basic Auth 플러그인을 기본 제공한다
- 동작 방식 :
    - HTTP Header(Authorization: Basic base64(username: password))를 통해 인증 데이터를 요청에 포함시켜 보낸다
    - 이때, username: password는 패스워드를 bcrypt 해싱한 다음 .htpasswd 파일에 저장된다
- Basic Auth 초기화 과정 :
    1. FF Core 설정파일에서 passwordfile 경로 읽기
    2. .htpasswd 파일 로딩
    3. 사용자 정보를 메모리에 저장
- Basic Auth 요청 인증 과정 :
    1. Authorization 헤더 확인
    2. Base64 디코딩을 통해 username: password 추출
    3. 메모리에서 사용자 조회하여 bcrypt 비교해서 일치여부 확

---

### 2.3 Basic Auth 특징

- passwd 파일은 FF 시작시 한 번만 읽기 때문에 사용자 추가 / 패스워드 변경 시에는 FF 재시작이 필요하다
- 인증 데이터는 메모리에 저장되어 처리 속도가 빠르다
- Basic Auth는 권한, 토큰, 세션 관리 기능을 제공하지 않고 단순히 사용자 인증만 수행한다

## 3 실습

### 3.1 Password 파일 생성

1. bcrypt password 파일 생성

```bash
touch test_users
```

1. htpasswd 설치

```bash
sudo apt update
```

```bash
sudo apt install apache2-utils
```

1. 사용자 생성

```bash
htpasswd -B test_users firefly
```

1. bcrypt password 파일 확인
- username: bcrypt hash(password) 형태가 출력돼야 한다

```bash
cat test_users
```

---

### 3.2 Namespace Auth 설정

1. FF config 수정

```bash
vi ~/.firefly/stacks/dev/runtime/config/firefly_core_0.yml
```

1. auth 플러그인 추가

```bash
# plugin 섹션
plugins:
  auth:
  - name: test_user_auth
    type: basic
    basic:
      passwordfile: /etc/firefly/test_users # docker 컨테이너 내부 경로
```

```bash
# namepace 섹션
namespaces:
  default: default
  predefined:
  - name: default
    description: Default namespace
    defaultKey: "<key>"
    plugins:
      - database0
      - blockchain0
      - dataexchange0
      - sharedstorage0
      - erc20_erc721
      - test_user_auth
```

1. docker 컨테이너에 Password 파일 마운트

```bash
vi ~/.firefly/stacks/dev/docker-compose.override.yml
```

```bash
services:
  firefly_core_0:
    volumes:
      - /home/hgene/.firefly/stacks/dev/test_users:/etc/firefly/test_users
```

1. FF 재시작
- Basic Auth는 초기화 시에만 password 파일을 읽는다

```bash
ff stop dev
```

```bash
ff start dev
```

---

### 3.3 Namespace Auth 테스트

- 인증없이 요청 전송 시, 결과는 Unauthorized

```bash
curl http://localhost:5000/api/v1/status
```

- 인증포함 요청 전송 시, 결과는 올바르게 나와야 함

```bash
curl -u "firefly:firefly" http://localhost:5000/api/v1/status
```

---

### 3.4 HTTP Listener Auth

1. FF config 수정

```bash
vi ~/.firefly/stacks/dev/runtime/config/firefly_core_0.yml
```

1. spi 섹션 추가

```bash
spi:
  address: 0.0.0.0
  enabled: true
  port: 5101
  publicURL: http://127.0.0.1:5101
  auth:
    type: basic
    basic:
      passwordfile: /etc/firefly/test_users
```

1. FF 재시작
- Basic Auth는 초기화 시에만 password 파일을 읽는다

```bash
ff stop dev
```

```bash
ff start dev
```

---

### 3.5 HTTP Listener Auth 테스트

- 인증없이 요청 전송 시, 결과는 Unauthorized

```bash
curl http://127.0.0.1:5101/spi/v1/namespaces
```

- 인증포함 요청 전송 시, 결과는 올바르게 나와야 함

```bash
curl -u "firefly:firefly" http://127.0.0.1:5101/spi/v1/namespaces
```