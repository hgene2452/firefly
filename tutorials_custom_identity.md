# 5. Tutorials (Custom Identity)

## 1 FireFly Custom Identity

[Create a custom identity - Hyperledger FireFly](https://hyperledger.github.io/firefly/latest/tutorials/create_custom_identity/)

### 1.1 FF 기본 Identity 구조

- FF 슈퍼노드(FF 인스턴스 전체)를 실행하면 기본적으로 2개의 Identity가 자동 생성된다
- Org Identity :
    - 이 슈퍼노드를 운영하는 조직을 대표하는 최상위 Identity
    - 멀티파티 네트워크에서 누가 이 노드를 운영하는지를 식별하는 루트
- Node Identity :
    - 이 슈퍼노드 안에서 실제로 메시지를 송수신/서명/이벤트 처리하는 노드 인스턴스를 대표
    - 네트워크 관점에서 어떤 노드가 보냈는지 구분 / 운영 주체(Org)와 실행 주체(Node)를 분리
- 하나의 슈퍼노드 내부에서 여러 개의 Custom Identity를 만들 수 있다

---

### 1.2 Custom Identity 필요성

- 현실에서는 조직이 하나의 노드를 운영한다는 것만으로 부족하고, 한 조직 안에도 다양한 행위 주체가 존재한다
cf. 고객별 지갑, 부서, 서비스 계정, 파트너 등
- 위의 사례들에서 슈퍼노드를 주체마다 새로 생성하면 운영비/복잡도 폭증과 같은 문제가 발생할 수 있다
- 따라서 FF는 아래와 같이 분리해서 운영할 수 있도록 제공한다
    - 네트워크 참여단위 = Org/슈퍼노드
    - 업무 행위 주체 = Custom Identity 여러개

---

### 1.3 Custom Identity 구성요소

- name : 사람/계정/역할을 나타내는 별칭
- key : 해당 Identity가 서명에 사용할 블록체인 키 (대개 EVM 주소, 개인키는 Signer에서 관리하고, FF는 주소를 참조한다)
- parent : 해당 Identity의 상위 소속 Identity
- did : FF가 Identity를 표현하는 내부 식별자 문자열

---

### 1.4 DID (Decentralized Identifier)

- 중앙기관이 아니라 분산 환경에서도 통용되는 식별자를 표준 형태로 표현한 것이다
- 기본 형태 : did:<method>:<identifier>

---

### 1.5 Identity 구조 계층

- FF Identity는 계층 구조로 모델링된다
- 멀티파티 네트워크에서는 같은 name이 다른 조직에도 있을 수 있는데, 이때는 parent가 다르면 충돌 없이 구분할 수 있다

```
Org Identity (루트)
  ├─ Node Identity (해당 노드 인스턴스)
  └─ Custom Identities (여러개)
        ├─ customer_001
        └─ merchant_A
```

## 2 Identity

[Identities - Hyperledger FireFly](https://hyperledger.github.io/firefly/latest/reference/identities/#verification)

### 2.1 FF Trust Model

- FF의 Trust Model은 데이터를 누가 작성했는가를 온체인 & 오프체인 두 레이어에서 동시에 검증하는 구조이다
- FF는 메시지나 트랜잭션이 발생할 때 다음을 확인해야 한다
    - 이 요청을 누가 보냈는가
    - 이 요청이 정말 해당 주체가 보낸 것이 맞는가
    
    이를 검증하기 위해 FF는 Identity + Verifier 구조를 사용한다
    
- Identity는 다음 두 가지를 연결한다
    - 요청의 주체 (DID)
    - 서명을 검증할 Verifier (블록체인 키 등)
- 키 관리 구조 : FF Core는 일반적으로 프라이빗 키를 직접 보관하지 않고, Signer 컴포넌트가 키를 보관하고 서명을 수행한다

```
FireFly Identity (org/custom/node)
   ↕
Verifier (검증 수단)
   ↕
Private Key (Signer가 보관)
```

---

### 2.2 Identity 타입

- org (organization) :
    - FF에서 가장 기본이 되는 루트 Identity이다
    - 역할 :
        - 멀티파티 네트워크에서 조직을 대표하는 Identity이다
        - 온체인 서명의 기본 주체이다
        - 모든 Identity 계층의 루트이다
    - 특징
        - Verifier는 블록체인 키이다
        - 멀티파티 네트워크에 참여하려면 반드시 root org를 먼저 생성해야 한다
        - org는 모든 타입의 자식 Identity를 가질 수 있다
    
    ```
    org
     ├ node
     ├ custom
     └ custom
    ```
    
    - 생성 API : POST /network/organizations/self
- node :
    - 오프체인 데이터 교환의 Identity이다
    - 역할 :
        - FF의 오프체인 메시지 송수신 주체이다
        - Data Exchange Connector(e.g. websocket)와 연결된다
    - 특징 :
        - node의 verifier는 블록체인 키가 아니라, Data Exchange 플러그인이 제공하는 식별자이다
        - node는 반드시 org의 자식이어야 한다
        - 하나의 org에는 여러 node가 있을 수 있다
        - node는 자식 Identity를 가질 수 없다
    
    ```
    org
     ├ node
     └ node
    ```
    
    - 생성 API : POST /network/nodes/self
- custom :
    - 애플리케이션이 정의하는 세분화된 Identity이다
    - 역할 :
        - 조직 내부의 사용자 / 서비스 / 역할 계정을 표현한다
    - 특징 :
        - org와 동일하게 온체인 verifier(블록체인 키)을 가진다
        - custom identity는 custom identity만 자식으로 가질 수 있다
    
    ```
    org
     └ custom
         └ custom
    ```
    

---

### 2.3 Messaging

- FF메시징은 오프체인 메시지를 보내지만, 온체인에 증거를 남겨서 위변조/부인 방지를 한다
- 따라서 메시지의 발신자는 반드시 온체인 Identity(누가 서명했는지?) & 오프체인 Identity(어떤 노드/커넥터로 보내왔나?) 모두 있어야 한다
- sender (author / key 규칙) :
    - FF는 author와 key 필드를 이용해 온체인 발신자(트랜잭션 서명자)를 결정한다
    
    | author | key | 의미 |
    | --- | --- | --- |
    | X | X | root org 사용 |
    | O | X | 해당 org/custom DID의 verifier를 key로 사용 |
    | X | O | key를 기준으로 reverse lookup하여 DID 결정 |
    | O | O | 그대로 사용 |
    - 오프체인 발신자는 항상 node의 이름과 데이터 교환 플러그인이 무엇인지에 따라 결정된다
- recipients :
    - 프라이빗 메시지에서는 수신자도 Identity와 Node를 가진다
    
    | Identity | Node | 의미 |
    | --- | --- | --- |
    | O | X | 해당 identity 또는 조상이 소유한 첫번째 node |
    | O | O | node는 반드시 해당 identity 또는 조상의 자식이어야 한다 |
    - receipient node는 오프체인 메시지 라우팅을 결정한다

---

### 2.4 시나리오

1. 가정
    - 2개의 조직이 FF 네트워크에 참여하고, 각 조직은 FF 슈퍼노드를 하나씩 운영한다

```
org: BankA
├ node: bankA-node1
├ custom: treasury
└ custom: compliance

org: BankB
└ node: bankB-node1
```

1. BankA가 BankB에게 메시지를 보낸다
    - 메시지 전송 바디 :

```json
{
  "author": "did:firefly:treasury",
  "recipients": [
    {
      "identity": "did:firefly:BankB"
    }
  ],
  "data": "Customer KYC verified"
}
```

1. Sender 결정
    - FF는 먼저 온체인 발신자(author/key)를 결정한다
        - author = treasury, key = X ⇒ author의 verifier 사용
        - treasury identity → verifier → Ethereum Address
2. 온체인 트랜잭션 생성
    - 위에서 추출된 Ethereum Address에 맵핑된 프라이빗키를 Signer에서 추출한다
    - 이후 추출한 프라이빗 키를 통해 트랜잭션 서명
3. 오프체인 메시지 전송
    - bankA-node1  → connector → bankB-node1
4. BankB FF node가 메시지를 받아 검증
    - 온체인 서명 검증 : author DID → verifier → Ethereum Address → Signer → private key
    - 오프체인 검증 : sender node + data exchange plugin

## 3 실습

> 해당 기능은 멀티파티 기능이기 때문에 실습을 위해 멀티파티로 설정을 변경한 후 원복한다
원복시, 생성한 identity(org, custom)는 DB에 등록되지만 비활성화된다
> 

### 3.1 새로운 계정 생성

- 새로운 계정 생성시, FF는 자동으로 signing container에 해당 키를 등록한다
- 하지만 아직 FF Identity에는 등록되지 않은 상태이다

```bash
ff accounts create dev
```

```bash
# 결과
{
	# Ethereum Address
  "address": "0x43a495d54427d31bcf87c355bea9e11c45c424ea",
  # 서명 키
  "privateKey": "29cdc3f3eb229fa42f8a33fcaeacbeffb4840da6134fdb7dfb7a43db461fed16",
  "ptmPublicKey": ""
}
```

---

### 3.2 Org UUID 조회

- Custom Identity를 생성하려면 parent가 될 org UUID가 필요하다
- 다만 싱글모드인 경우, org는 암묵적으로만 존재하고 identity registry에 등록되지 않는다
따라서 아래의 API 응답값에 org가 포함되지 않는다

```bash
Supernode
   └ implicit org
        └ custom identities
```

```bash
curl -u "firefly:firefly" http://localhost:5000/api/v1/status
```

---

### 3.3 Custom Identity 등록

- 싱글파티인 경우, 위에서 identity registry에 등록된 org가 없기 때문에 필수값인 parent 값이 없어 custom identity를 등록할 수 없다

```bash
curl -u "firefly:firefly" \
  -X POST "http://localhost:5000/api/v1/identities" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test-custom-identity",
    "key": "<3.1 결과 address값>"
    "parent": "<3.2 결과 org.id 값>"
  }'
```

---

### 3.4 Custom Identity 등록 확인

```bash
curl -u "firefly:firefly" http://localhost:5000/api/v1/identities?fetchverifiers=true
```