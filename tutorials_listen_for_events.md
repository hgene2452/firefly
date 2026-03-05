# 6. Tutorials (Listen for events)

## 1 Event-Driven 아키텍처

[Listen for events - Hyperledger FireFly](https://hyperledger.github.io/firefly/latest/tutorials/events/)

### 1.1 이벤트 기반 시스템

- FF는 Event-Driven 아키텍처 기반으로 동작하는 Web3 오케스트레이션 플랫폼이다
- 즉, 시스템의 흐름이 요청이 아닌 이벤트를 중심으로 처리된다
- 일반적인 웹 서비스는 다음과 같은 요청 기반 구조를 가진다 :
    - Client → API → DB → Response
    - 이 구조에서는 클라이언트의 요청이 시스템 흐름의 시작점이 된다
- 반면 FF는 다음과 같은 이벤트 기반 구조를 가진다 :
    - Action → Event → Processing
    - 어떤 행동(Action)이 발생하면 시스템에서 이벤트(Event)가 생성되고, 다른 시스템이나 애플리케이션이 해당 이벤트를 받아 후속 처리(Processing)를 수행한다
- 이 구조는 특히 다음과 같은 멀티파티 Web3 시스템에 적합하다
    - 여러 조직 간의 데이터 교환
    - 블록체인 트랜잭션 상태 변화
    - 스마트 컨트랙트 이벤트
    - 메시지 브로드캐스트

---

### 1.2 Private History

- FF는 각 노드별로 독립적인 Private History를 유지한다
- 이는 단순한 블록체인 데이터가 아니라 FF 노드가 관찰하고 처리한 모든 활동 기록이다
- 저장되는 데이터 :
    - 온체인 트랜잭션 : 블록체인에 기록된 모든 트랜잭션
    - 오프체인 메시지 : 블록체인 외부 메시지 교환 관리
    - 이벤트 기록 : 시스템 내 모든 이벤트
- 이 기록을 통해 조직은 자신의 노드 기준 활동 기록을 확인할 수 있다

---

### 1.3 FireFly Event 전송 구조

- FF에서 생성된 이벤트는 Event Transport 플러그인을 통해 애플리케이션에 전달된다
- FF는 이벤트 전달 방식을 플러그인 구조로 설계하여 다양한 시스템과 연동할 수 있다
- 대표적인 방식 :
    
    
    | 방식 | 설명 |
    | --- | --- |
    | WebSocket | 실시간 이벤트 스트림 |
    | Webhook | HTTP 기반 이벤트 전달 |
    | Kafka | 메시지 스트리밍 시스템 연동 |
    | Custom Plugin | 기업 시스템 연동 가능 |

## 2 WebSocket 이벤트 구독 방식

### 2.1 WebSocket

- 가장 일반적인 방식으로 실시간 이벤트 스트리밍을 제공한다
- 특징 :
    - 지속 연결
    - 낮은 Latency
    - 실시간 이벤트 수신
- 구조 : FireFly → WebSocket → Application Listener
- 주로 다음 용도로 사용된다 :
    - 실시간 UI 업데이트
    - 트랜잭션 상태 추적
    - 이벤트 기반 비즈니스 로직 실행

---

### 2.2 WebSocket Ephemeral Subscription

- Ephemeral Subscription은 임시 이벤트 리스너이다
- 즉, 애플리케이션이 연결되어 있는 동안에만 이벤트를 수신한다
- FF는 이 연결을 애플리케이션 sibscription으로 저장하지 않고, 연결 종료시 구독 상태를 삭제한
- 주로 다음 용도로 사용된다 :
    - UI
    - 테스트
    - 디버깅
    - 간단한 이벤트 확인
- 연결 URL :
    
    ```markdown
    ws://localhost:5000/ws?namespace=default&ephemeral&autoack&filter.events=message_confirmed
    ```
    
    - namespace : FF namespace
    - ephemeral : 연결 동안만 이벤트 수신
    - autoack : FF가 자동으로 ACK 전송
    - filter.events : 특정 이벤트만 수신

---

### 2.3 Durable Subscription

- Durable Subscription은 운영 환경에서 사용하는 안정적인 이벤트 소비 방식이다
- Ephemeral subscription과 달리 FF가 이벤트 소비 상태를 기록한다
- FF는 다음의 정보를 저장한다 :
    - 마지막으로 소비된 이벤트
    - 구독 애플리케이션 정보
    - 이벤트 처리 상태
- 생성 :
    
    ```json
    POST /namespaces/default/subscriptions
    
    {
      "transport": "websockets",
      "name": "app1",
      "options": {
        "firstEvent": "newest",
        "readAhead": 50
      }
    }
    ```
    
    - transport : 이벤트 전송 방식
    - name : subscription 이름
    - firstEvent : 이벤트 시작 위치
    - readAhead : 이벤트 버퍼 크기
- Durable subscription에서는 애플리케이션이 ACK를 보내야 한다

---

### 2.4 Event Payload 구조

- FF WebSocket 이벤트는 전체 메시지 데이터를 포함하지 않는다
- 대신 이벤트에 대한 메타데이터만 전달한다
    - WebSocket 메시지 크기 최소화
    - 네트워크 효율성
    - 필요한 데이터만 선택적으로 조회
    
    위의 이유들 때문이다
    
- Event 페이로드 예시 :
    
    ```json
    {
      "id": "8f0da4d7-8af7-48da-912d-187979bf60ed",
      "sequence": 61,
      "type": "message_confirmed",
      "namespace": "default",
      "reference": "9710a350-0ba1-43c6-90fc-352131ce818a", // 관련 메시지 ID
      "created": "2021-07-02T04:37:47.6556589Z"
    }
    ```
    
- 이벤트에는 메시지의 기본 정보가 포함될 수 있다 :
    
    ```json
    "message": {
      "header": {
        "id": "9710a350",
        "type": "broadcast",
        "author": "0x1d14b65d"
      }
    }
    ```
    
- 이벤트에 포함되지 않은 데이터는 REST API로 조회할 수 있다
    
    ```bash
    # 전체 메시지 조회
    GET /api/v1/namespaces/default/messages/{id}?data=true
    
    # 데이터 배열만 조회
    GET /api/v1/namespaces/default/messages/{id}/data
    ```
    

## 3 실습

### 3.1 WebSocket 테스트 툴 준비

```bash
npm install -g wscat
```

```bash
wscat --version
```

---

### 3.2 Ephemeral Websocket 테스트

1. Ephemeral Websocket 연결

```bash
wscat -c "ws://localhost:5000/ws?namespace=default&ephemeral&autoack"
```

1. 이벤트 발생시키기
- MasterMinter API 요청해서 트랜잭션 이벤트 발생시켜본다

```bash
curl -u firefly:firefly -X POST \
"http://127.0.0.1:5000/api/v1/namespaces/default/apis/masterMinter/invoke/configureController?confirm=true" \
-H "Content-Type: application/json" \
-d '{
  "input": {
    "_controller": "0x5c342e04a87d3f7991dc4ecab48cdc35011b4755",
    "_worker": "0x5c342e04a87d3f7991dc4ecab48cdc35011b4755"
  }
}'
```

1. WebSocket에서 이벤트 확인
- 성공시, WebSocket 창에서 아래의 이벤트가 확인되어야 한다

```bash
{
    "id": "063223a7-a806-4cf4-bff5-f11b37420654",
    "sequence": 60,
    "type": "transaction_submitted",
    "namespace": "default",
    "reference": "ec61ca3e-fc8c-4252-bccd-7ed69e466bd9",
    "tx": "ec61ca3e-fc8c-4252-bccd-7ed69e466bd9",
    "topic": "contract_invoke",
    "created": "2026-03-05T02:09:57.087184401Z",
    "transaction": {
        "id": "ec61ca3e-fc8c-4252-bccd-7ed69e466bd9",
        "namespace": "default",
        "type": "contract_invoke",
        "created": "2026-03-05T02:09:57.07987427Z"
    },
    "subscription": {
        "id": "4141809c-c156-439d-9ee0-78576de23359",
        "namespace": "default",
        "name": "4141809c-c156-439d-9ee0-78576de23359"
    }
}
...
```

---

### 3.3 REST API로 트랜잭션 데이터 조회

1. 트랜잭션 상세 조회

```bash
curl -u firefly:firefly \
"http://127.0.0.1:5000/api/v1/namespaces/default/transactions/ec61ca3e-fc8c-4252-bccd-7ed69e466bd9"

{
    "id": "ec61ca3e-fc8c-4252-bccd-7ed69e466bd9",
    "namespace": "default",
    "type": "contract_invoke",
    "created": "2026-03-05T02:09:57.07987427Z"
}
```

1. 트랜잭션 operation 리스트 조회

```bash
curl -u firefly:firefly \
"http://127.0.0.1:5000/api/v1/namespaces/default/transactions/ec61ca3e-fc8c-4252-bccd-7ed69e466bd9/operations"
```

1. 특정 operation 조회

```bash
curl -u firefly:firefly \
"http://127.0.0.1:5000/api/v1/namespaces/default/operations/4e436fa0-2bf7-450e-b15a-586ae6e29090"
```

---

### 3.4 Durable Subscription 테스트

1. Subscription 생성

```bash
curl -u firefly:firefly \
-X POST http://127.0.0.1:5000/api/v1/namespaces/default/subscriptions \
-H "Content-Type: application/json" \
-d '{
  "transport": "websockets",
  "name": "app1",
  "options": {
    "firstEvent": "newest",
    "readAhead": 50
  }
}'
```

- 성공시, 아래와 같이 결과가 출력된다

```bash
{
    "id": "263edf1c-18c2-467f-a370-9839533f9320",
    "namespace": "default",
    "name": "app1",
    "transport": "websockets",
    "filter": {
        "message": {},
        "transaction": {},
        "blockchainevent": {}
    },
    "options": {
        "firstEvent": "61",
        "readAhead": 50,
        "withData": false
    },
    "created": "2026-03-05T02:19:43.455170228Z",
    "updated": null
}
```

1. Durable Websocket 연결

```bash
wscat -c "ws://localhost:5000/ws?namespace=default&name=app1"
```

1. 트랜잭션 이벤트 발생시키기

```bash
curl -u firefly:firefly -X POST \
"http://127.0.0.1:5000/api/v1/namespaces/default/apis/masterMinter/invoke/configureController?confirm=true" \
-H "Content-Type: application/json" \
-d '{
  "input": {
    "_controller": "0x5c342e04a87d3f7991dc4ecab48cdc35011b4755",
    "_worker": "0x5c342e04a87d3f7991dc4ecab48cdc35011b4755"
  }
}'
```

1. WebSocket에서 이벤트 확인
- 성공시, WebSocket 창에서 아래의 이벤트가 확인되어야 한다

```bash
{
    "id": "c56a4a01-f4a5-47b4-b6df-dcacfa5d96cd",
    "sequence": 64,
    "type": "transaction_submitted",
    "namespace": "default",
    "reference": "8d901776-71cc-419f-8b80-e2a9c373c2c2",
    "tx": "8d901776-71cc-419f-8b80-e2a9c373c2c2",
    "topic": "contract_invoke",
    "created": "2026-03-05T02:22:17.756658406Z",
    "transaction": {
        "id": "8d901776-71cc-419f-8b80-e2a9c373c2c2",
        "namespace": "default",
        "type": "contract_invoke",
        "created": "2026-03-05T02:22:17.755525721Z"
    },
    "subscription": {
        "id": "263edf1c-18c2-467f-a370-9839533f9320",
        "namespace": "default",
        "name": "app1"
    }
}
```

1. 이후 Websocket 창에서 ACK를 해주지 않으면 다음 이벤트 전달이 멈출 수 있다
- 이때,  ACK 대상은 도착한 이벤트 전체이다 (3개가 왔으면 3개 모두 ACK 처리 해줘야 한다)
- 도착한 이벤트에 대한 페이로드 조회는 3.3과 동일하게 처리 가능하다

```bash
{ "type": "ack", "id": "617db63-2cf5-4fa3-8320-46150cbb5372" }
```