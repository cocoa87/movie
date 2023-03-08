![image](https://user-images.githubusercontent.com/48465481/223025551-99070f96-0ac5-4863-a409-34b5e0013394.png)

# 예제 - Movie Reservation

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 영화예약](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#2.-CQRS-:-명령과 쿼리 분리)
    - [폴리글랏 퍼시스턴스](#1.-Saga-(Pub-Sub))
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

영화예매 커버하기 - https://blog.naver.com/2725kya/222612662750

기능적 요구사항
1. 영화관이 상영할 영화를 등록/수정/삭제한다.
2. 고객이 영화를 선택하여 예약한다.
3. 예약과 동시에 결제가 진행된다.
4. 예약이 완료되면 예약 내역(Message)이 전달된다.
5. 고객이 예약을 취소할 수 있다.
6. 예약 사항이 취소될 경우 취소 내역(Message)이 전달된다.
7. 영화관에 후기(review)를 남길 수 있다.
8. 전체적인 영화에 대한 정보를 한 화면에서 확인 할 수 있다.(viewpage)

비기능적 요구사항
1. 트랜잭션
- 결제가 되지 않은 예약 건은 성립되지 않아야 한다. (Sync 호출)
2. 장애격리
- 영화 등록 및 메시지 전송 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
- 예약 시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시 후에 하도록 유도한다 Circuit breaker, fallback
3. 성능
- 모든 영화에 대한 정보 및 예약 상태 등을 한번에 확인할 수 있어야 한다 (CQRS)
- 예약의 상태가 바뀔 때마다 메시지로 알림을 줄 수 있어야 한다 (Event driven)


# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  https://labs.msaez.io/#/storming/b8a9220f305f4adddc58aa6e81e80a04


### 이벤트 도출 완료
![부적격이벤트탈락](https://user-images.githubusercontent.com/48465481/223027186-1feeb7fa-7a31-4e67-9684-c3b4fcd75e1a.JPG)

### 완성모형
![image](https://user-images.githubusercontent.com/48465481/223027507-874b4fee-14b3-457f-8fb8-428602499db5.png)



# 구현:

## 1. Saga (Pub-Sub)
 - 영화예매 취소 가능
 - 예매 취소시 해당 좌석이 다시 예약 가능한 상태가 됨
 - 예약이 취소되면 재고량이 증가한다
## 2. CQRS : 명령과 쿼리 분리
 - 고객은 현재 예매 상태를 언제든 확인 가능해야함
 - : View의 CQRS 복붙 및 설명
## 3. Compensation & Correlation 
 -어떠한 이벤트로 인하여 발생한 변경사항들에 대하여 고객이 원하거나 어떠한 기술적 이유로 인하여 해당 트랜잭션을 취소해야 하는 경우 이를 원복하거나 보상해주는 처리를 Compensation 이라고 한다. 그리고 해당 취소건에 대하여 여러개의 마이크로 서비스 내의 데이터간 상관 관계를 키값으로 연결하여 취소해야 하는데, 이러한 관계값에 대한 처리를 Correlation 이라고 한다.
 - 예매 취소시 다른 것들 모두 원복하기
 - : 리뷰생성/삭제에 따른 리뷰Cnt 변화 
## 6. Gateway / Ingress
 - API Gateway를 사용하여 마이크로 서비스들의 엔드포인트 단일화
 - 주문, 상품, 배송 서비스를 분기하는 라우팅 룰을 가진 Ingress 를 생성한다
 - Ingress 는 Kubernetes 의 스펙일 뿐, 이를 실질적으로 지원하는 ingress controller 가 필요하기 때문이다. 다행히, 우리에겐 무료로 사용할 수 있는 nginx 인그레스 프로바이더를 사용할 수 있다.
## 7. Deploy / Pipeline
 -
## 8. Autoscale (HPA)
 - 클라우드의 리소스를 잘 활용하기 위해서는 요청이 적을때는 최소한의 Pod 를 유지한 후에 요청이 많아질 경우 Pod를 확장하여 요청을 처리할 수 있다.
 - Auto Scale-Out 실습 (hpa: HorizontalPodAutoscaler 설정)
## 9. Zero-downtime deploy (Readiness probe)
 - 배포시 다운타임의 존재 여부를 확인하기 위하여, siege 라는 부하 테스트 툴을 사용한다.
 - Kafka 가 설치되어있어야 한다
## 10. Persistence Volume/ConfigMap/Secret
 - 파일시스템 (볼륨) 연결과 데이터베이스 설정
## 11. Self-healing (liveness probe)

 kubectl describe 커맨드로 Pod 이벤트의 메시지 변화를 확인한다.
kubectl describe po liveness-exec
HttpGet ProbeAction
HttpGet type의 Probe Action이 설정된 주문 마이크로서비스 배포
kubectl apply -f https://raw.githubusercontent.com/acmexii/demo/master/edu/order-liveness.yaml
배포된 주문서비스에 대해 라우터를 생성한다.
kubectl expose deploy order --type=LoadBalancer --port=8080
kubectl get svc
Order Liveness Probe를 명시적으로 Fail 상태로 전환한다.
# Liveness Probe 확인
```
http EXTERNAL-IP:8080/actuator/health
```
- Liveness Probe Fail 설정 및 확인
```
http put EXTERNAL-IP:8080/actuator/down
http EXTERNAL-IP:8080/actuator/health
```
- Probe Fail에 따른 쿠버네티스 동작확인
```
kubectl get pod
kubectl describe pod/[ORDER-POD객체]
```

## 12. Apply Service Mesh
 - 트래픽 제어? 분산?
## 13. Loggregation / Monitoring
 - 마이크로서비스 통합 로깅 with EFK stack
 - MSA 모니터링 with installing Grafana

## 여기까지

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
각 서비스마다
mvn spring-boot:run
```



## 
1. Saga (Pub-Sub)
 - 영화예매 취소 가능
 - 예매 취소시 해당 좌석이 다시 예약 가능한 상태가 됨
 - 예약이 취소되면 재고량이 증가한다
2. CQRS : 명령과 쿼리 분리
 - 고객은 현재 예매 상태를 언제든 확인 가능해야함
3. Compensation & Correlation :어떠한 이벤트로 인하여 발생한 변경사항들에 대하여 고객이 원하거나 어떠한 기술적 이유로 인하여 해당 트랜잭션을 취소해야 하는 경우 이를 원복하거나 보상해주는 처리를 Compensation 이라고 한다. 그리고 해당 취소건에 대하여 여러개의 마이크로 서비스 내의 데이터간 상관 관계를 키값으로 연결하여 취소해야 하는데, 이러한 관계값에 대한 처리를 Correlation 이라고 한다.
 - 예매 취소시 다른 것들 모두 원복하기
6. Gateway / Ingress
 - API Gateway를 사용하여 마이크로 서비스들의 엔드포인트 단일화
 - 주문, 상품, 배송 서비스를 분기하는 라우팅 룰을 가진 Ingress 를 생성한다
 - Ingress 는 Kubernetes 의 스펙일 뿐, 이를 실질적으로 지원하는 ingress controller 가 필요하기 때문이다. 다행히, 우리에겐 무료로 사용할 수 있는 nginx 인그레스 프로바이더를 사용할 수 있다.
7. Deploy / Pipeline
 -
8. Autoscale (HPA)
 - 클라우드의 리소스를 잘 활용하기 위해서는 요청이 적을때는 최소한의 Pod 를 유지한 후에 요청이 많아질 경우 Pod를 확장하여 요청을 처리할 수 있다.
 - Auto Scale-Out 실습 (hpa: HorizontalPodAutoscaler 설정)
9. Zero-downtime deploy (Readiness probe)
 - 배포시 다운타임의 존재 여부를 확인하기 위하여, siege 라는 부하 테스트 툴을 사용한다.
 - Kafka 가 설치되어있어야 한다
10. Persistence Volume/ConfigMap/Secret
 - 파일시스템 (볼륨) 연결과 데이터베이스 설정
11. Self-healing (liveness probe)
 - 셀프힐링 실습 (livenessProbe)
12. Apply Service Mesh
 - 트래픽 제어? 분산?
13. Loggregation / Monitoring
 - 마이크로서비스 통합 로깅 with EFK stack
 - MSA 모니터링 with installing Grafana

