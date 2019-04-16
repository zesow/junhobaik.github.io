## Apache Kafka 란

## Kafka 구조
### producer
### consumer

## 예제에 적용

### 흐름
- (목표)특정 서비스에서 이벤트를 하나 발생시키면 이벤트가 Agent를 거쳐 목표 서비스로 도착하는 과정을, post REST 주소와 JSON 객체를 입력하는 순간 일어나는 일부터 순서대로 설명한다.

[process service]
- http://localhost:8081/process 와 process entity에 맞는 JSON 객체를 입력한다.
- (ProcessResultResource) register 메서드에서 ProcessResultService.register 로 전달
- (ProcessResultLogic) register 메서드에서 agentProxy.sendProcess 로 전달
- (AgentDelegator) sendProcess 메서드에서 AgentClient.sendProcess로 전달.

---

[Agent Service]
- (AgentClient) sendProcess로 전달된 파라미터를 여기서 Agent의 Kafka 발행 REST 주소로 맵핑해서 전달함. --> http://localhost:9710/agent/message
- (AgentResource) sendProcess 메서드에 다시 맵핑되어서, 이번엔 Logic이 아닌 SendEvent가 실행됨.

(service - producer)
- (SendEvent) send 메서드에서 SourceBinding.outputChannel()실행
- (SourceBinding) outPutChannel() 로 Kafka 에 메시지를 보냄.
 
---

[payment service]

(service - consumer)
 