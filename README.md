# 개인과제 - 12번가

![12번가](https://user-images.githubusercontent.com/88864433/133467597-709524b1-4613-4dab-bc57-948f433ad565.png)
---------------------------------

# Table of contents

- [조별과제 - 12번가 배송서비스](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [DDD 의 적용](#DDD의-적용)
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency) 
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [API 게이트웨이](#API-게이트웨이)
  - [운영](#운영)
    - [Deploy/Pipeline](#deploypipeline)
    - [동기식 호출 / Circuit Breaker / 장애격리](#동기식-호출-circuit-breaker-장애격리)
    - [Autoscale (HPA)](#autoscale-(hpa))
    - [Zero-downtime deploy (Readiness Probe)](#zerodowntime-deploy-(readiness-Probe))
    - [Self-healing (Liveness Probe)](#self-healing-(liveness-probe))
    - [운영유연성](#운영유연성)



# 서비스 시나리오
	
[서비스 아키텍쳐]
주문팀, 상품배송팀, 마케팅팀

[서비스 시나리오]

가. 기능적 요구사항

1. [주문팀]고객이 상품을 선택하여 주문 및 결제 한다.

2. [주문팀]주문이 되면 주문 내역이 상품배송팀에게 전달된다. orderplaced (Pub/sub)

3. [상품배송팀] 주문을 확인하고 쿠폰 발행을 요청한다. ( Req/Res )

4. [마케팅팀] 쿠폰을 발행하고 상품배송팀에 알린다. ( Req/Res )

5. [상품배송팀] 쿠폰이 발행이 완료되면(반드시), 배송을 출발한다.

6. [주문팀]고객이 주문을 취소한다.

7. [주문팀]주문 취소를 상품배송팀에 알린다.

8. [상품배송팀] 주문 취소를 확인하고 쿠폰 취소를 요청한다. ( Req/Res )

9. [마케팅팀] 발행된 쿠폰을 취소하고 상품배송팀에 알린다. ( Req/Res )

10. [상품배송팀] 쿠폰이 발행이 취소되면(반드시), 배송을 취소한다.


나. 비기능적 요구사항

1. [설계/구현]Req/Resp : 쿠폰이 발행된 건에 한하여 배송을 시작한다. 

2. [설계/구현]CQRS : 고객이 주문상태를 확인 가능해야한다.

3. [설계/구현]Correlation : 주문을 취소하면 -> 쿠폰을 취소하고 -> 배달을 취소 후 주문 상태 변경

4. [설계/구현]saga : 서비스(상품팀, 상품배송팀, 마케팅팀)는 단일 서비스 내의 데이터를 처리하고, 각자의 이벤트를 발행하면 연관된 서비스에서 이벤트에 반응하여 각자의 데이터를 변경시킨다.

5. [설계/구현/운영]circuit breaker : 배송 요청 건수가 임계치 이상 발생할 경우 Circuit Breaker 가 발동된다. 

다. 기타 

1. [설계/구현/운영]polyglot : 상품팀과 주문팀은 서로 다른 DB를 사용하여 polyglot을 충족시킨다.


# 추가 시나리오
가. 기능적 요구사항

### 11. [전화주문] 고객이 전화를 통해 상품을 주문 및 결제한다. 

### 12. [전화주문] 전화로 주문이 완료되면 자동으로 상품배송팀에 정보가 전달된다. callorderplaced (Pub/sub)

### 13. [전화주문] 고객이 전화를 통해 주문을 취소한다. 

### 14. [전화주문] 전화를 통한 주문취소를 상품배송팀에 알린다. 

나. 비기능적 요구사항

1. [설계/구현]CQRS : 고객이 주문상태를 확인 가능해야한다.
### - 기존주문과 전화주문을 포함하여 고객은 주문상태를 확인 가능해야 한다. 

2. [설계/구현]Correlation : 주문을 취소하면 -> 쿠폰을 취소하고 -> 배달을 취소 후 주문 상태 변경
### - 전화 주문 취소시에도 쿠폰을 취소하고, 배달을 취소 후에 상태를 변경한다. 

3. [설계/구현]saga : 서비스(상품팀, 상품배송팀, 마케팅팀)는 단일 서비스 내의 데이터를 처리하고, 각자의 이벤트를 발행하면 연관된 서비스에서 이벤트에 반응하여 각자의 데이터를 변경시킨다.
### - 주문팀에서와 동일하게 전화주문팀에서 데이터가 변경되었을 때도 데이터가 변경되어야 한다.

다. 기타

1. [설계/구현/운영]polyglot : 상품팀과 주문팀은 서로 다른 DB를 사용하여 polyglot을 충족시킨다.
### - 주문팀과 전화주문팀은 동일한 DB를 사용한다.



# 체크포인트

+ 분석 설계

	- 이벤트스토밍:
		- 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
		- 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
		- 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
		- 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?
	- 서브 도메인, 바운디드 컨텍스트 분리
		- 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
			- 적어도 3개 이상 서비스 분리
		- 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
		- 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
	- 컨텍스트 매핑 / 이벤트 드리븐 아키텍처
		- 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
		- Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
		- 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
		- 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
		- 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?
	- 헥사고날 아키텍처
		- 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?

- 구현

	- [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
		-Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
		- [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
		- 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
	- Request-Response 방식의 서비스 중심 아키텍처 구현
		- 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
		- 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
	- 이벤트 드리븐 아키텍처의 구현
		- 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
		- Correlation-key: 각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
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
		- Contract Test : 자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?
---

# 분석/설계

## Event Stoming 결과

- MSAEz로 모델링한 이벤트스토밍 결과
https://www.msaez.io/#/storming/7znb05057kPWQo1TAWCkGM0O2LJ3/5843d1078a788a01aa837bc508a68029


### 이벤트 도출

![1_EVENTSTOMING결과](https://user-images.githubusercontent.com/88864433/134791183-9d8f81f7-49a3-4326-8636-ab0d697dc4d4.PNG)

```
1차적으로 필요하다고 생각되는 이벤트를 도출하였다 
``` 

### 부적격 이벤트 탈락

![2_부적격이벤트탈락](https://user-images.githubusercontent.com/88864433/134791192-8c11ea8f-df09-4122-b97d-2906bd4a6d7a.PNG)


```
- 과정 중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
- ‘재고가 충족됨’, ‘재고가 부족함’ 은 배송 이벤트를 수행하기 위한 내용에 가까우므로 이벤트에서 제외
- 주문과 결제는 동시에 이루어진다고 봐서 주문과 결제를 묶음
```
#### - 전화주문을 확인하는 과정은 필요없다고 생각하여 탈락하고, 오더와 동일하게 전화주문과 결제는 묶음. 


![3_탈락후결과](https://user-images.githubusercontent.com/88864433/134791195-ade44b8a-470c-4c09-99b4-bc769647e464.PNG)


### 액터, 커맨드를 부착하여 읽기 좋게 

![4_액터커맨드를 부착하여 읽기 좋게](https://user-images.githubusercontent.com/88864433/135371696-9649d76f-0bec-4f1d-bebe-22d21ef18a08.PNG)


 
### 어그리게잇으로 묶기

![5_어그리게잇으로 묶기](https://user-images.githubusercontent.com/88864433/135371739-de743d91-5f78-479e-9dd4-f46e0e6f1804.PNG)

 
``` 
- 고객의 주문후 배송팀의 배송관리, 마케팅의 쿠폰관리는 command와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어줌
```
##### - 전화주문의 경우도 일반주문과 동일하게 배송팀의 배송관리, 마케팅의 쿠폰관리에 의해 트랜잭션이 유지되어야 한다. 


### 바운디드 컨텍스트로 묶기

![6_바운디드컨텍스트로묶기](https://user-images.githubusercontent.com/88864433/135371751-092320d3-caa2-4905-99f7-1c21ca3bbc65.PNG)

 
```
- 도메인 서열 분리 
    #####- Core Domain:  order, delivery, callorder : 없어서는 안될 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 order의 경우 1주일 1회 미만, delivery의 경우 1개월 1회 미만
    - Supporting Domain:  marketing : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함
```
### 폴리시 부착

![8 폴리시부착](https://user-images.githubusercontent.com/88864433/135371808-a0055f54-9a65-4347-96f1-81ef5c28656c.PNG)
 

### 폴리시의 이동과 컨텍스트 맵핑 (점선은 Pub/Sub, 실선은 Req/Resp) 

![8-1 폴리시의 이동과 컨텍스트 맵핑](https://user-images.githubusercontent.com/88864433/135371836-d2fc4ef7-09f2-4e50-82f5-60d289f4995d.PNG)

 

### 완성된 모형

![MSAEZ_1](https://user-images.githubusercontent.com/88864433/134791220-0258016c-87dd-4b73-b0a7-7fcb89942cde.PNG)

 
### 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![9_주문에대한검증](https://user-images.githubusercontent.com/88864433/134791226-dde1c98c-d913-453f-9b2d-31d075911ea2.PNG)



##### - 고객이 전화를 통해 주문하고 결제한다 (ok)
##### - 결제가 완료되면 주문 내역이 배송팀에 전달된다. (ok)
##### - 마케팅팀에서 쿠폰을 발행한다 (ok)
##### - 쿠폰이 발행된 것을 확인하고 배송을 시작한다 (ok)

![10_주문취소에대한검증](https://user-images.githubusercontent.com/88864433/134791230-c1b500e0-11ff-49fe-a28f-644e0d0300b8.PNG)


 
##### - 고객이 전화로 주문을 취소할 수 있다 (ok)
##### - 전화로 주문을 취소하면 결제도 함께 취소된다 (ok)
##### - 전화로 주문이 취소되면 배송팀에 전달된다 (ok)
##### - 마케팅팀에서 쿠폰발행을 취소한다 (ok)
##### - 쿠폰발행이 취소되면 배송팀에서 배송을 취소한다 (ok)


### 비기능 요구사항에 대한 검증 

![11_비기능적요구사항검증](https://user-images.githubusercontent.com/88864433/134791233-461f4fb3-c6d8-4a97-9fb5-a9722e283a03.PNG)


```
1. [설계/구현]CQRS : 고객이 주문상태를 확인 가능해야한다.
2. [설계/구현]Correlation : 고객이 전화로 주문을 취소하면 -> 쿠폰을 취소하고 -> 배달을 취소 후 주문 상태 변경
3. [설계/구현]saga : 서비스(전화상품팀, 상품팀, 상품배송팀, 마케팅팀)는 단일 서비스 내의 데이터를 처리하고, 각자의 이벤트를 발행하면 연관된 서비스에서 이벤트에 반응하여 각자의 데이터를 변경시킨다.
``` 

### 헥사고날 아키텍처 다이어그램 도출 (업데이트 필요) 

![12_헥사고날](https://user-images.githubusercontent.com/88864433/135393741-d717527b-f96c-4fe7-866b-53c6f69cd638.PNG)
 

```
- Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
- 호출관계에서 PubSub 과 Req/Resp 를 구분함
- 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
```

# 구현

- 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 바운더리 컨텍스트 별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd order
mvn spring-boot:run

cd productdelivery 
mvn spring-boot:run

cd marketing
mvn spring-boot:run 

cd callorder
mvn spring-bott:run

```

- AWS 클라우드의 EKS 서비스 내에 서비스를 모두 배포하였다. 
```
root@labs--868582820:/home/project# kubectl get all
NAME                                   READY   STATUS    RESTARTS   AGE
pod/callorder-9d9b9cfcf-vr56v          1/1     Running   0          30m
pod/gateway-5c8fdfdf-nl7cb             1/1     Running   0          6h16m
pod/marketing-55d7d8658d-j5djz         1/1     Running   7          5h48m
pod/order-77d888fd99-vmkv8             1/1     Running   0          127m
pod/orderstatus-6d54bf687d-q5jvh       1/1     Running   7          5h48m
pod/productdelivery-666758dcf5-fbshf   1/1     Running   7          5h48m
pod/siege                              1/1     Running   0          5h26m

NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)          AGE
service/callorder         ClusterIP      10.100.150.45    <none>                                                                      8080/TCP         29m
service/gateway           LoadBalancer   10.100.89.144    aba2e181c0a58401ca3ea7c74bf798e5-725324256.ca-central-1.elb.amazonaws.com   8080:32321/TCP   6h9m
service/kubernetes        ClusterIP      10.100.0.1       <none>                                                                      443/TCP          8h
service/marketing         ClusterIP      10.100.21.247    <none>                                                                      8080/TCP         5h48m
service/order             ClusterIP      10.100.21.110    <none>                                                                      8080/TCP         127m
service/orderstatus       ClusterIP      10.100.153.201   <none>                                                                      8080/TCP         5h48m
service/productdelivery   ClusterIP      10.100.8.67      <none>                                                                      8080/TCP         5h48m

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/callorder         1/1     1            1           30m
deployment.apps/gateway           1/1     1            1           6h16m
deployment.apps/marketing         1/1     1            1           5h48m
deployment.apps/order             1/1     1            1           127m
deployment.apps/orderstatus       1/1     1            1           5h48m
deployment.apps/productdelivery   1/1     1            1           5h48m

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/callorder-9d9b9cfcf          1         1         1       30m
replicaset.apps/gateway-5c8fdfdf             1         1         1       6h16m
replicaset.apps/marketing-55d7d8658d         1         1         1       5h48m
replicaset.apps/order-77d888fd99             1         1         1       127m
replicaset.apps/orderstatus-6d54bf687d       1         1         1       5h48m
replicaset.apps/productdelivery-666758dcf5   1         1         1       5h48m
```


# DDD의 적용
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가? 

각 서비스 내에 도출된 핵심 Aggregate Root 객체를 Entity로 선언하였다. (주문(order), 배송(productdelivery), 마케팅(marketing)) 
##### 전화주문의 경우는 크게 주문의 범주에 들어가므로 주문의 Entity와 동일하게 사용하나 소스는 따로 구현한다.

주문 Entity (CallOrder.java) 
```
@Entity
@Table(name="Order_table")
public class Order {

    
    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String username;
    private String address;
    private String phoneNo;
    private String productId;
    private int qty; //type change
    private String payStatus;
    private String userId;
    private String orderStatus;
    private Date orderDate;
    private String productName;
    private Long productPrice;
    private String couponId;
    private String couponKind;
    private String couponUseYn;

    @PostPersist
    public void onPostPersist(){
    	
         Logger logger = LoggerFactory.getLogger(this.getClass());

    	
        CallOrderPlaced callorderPlaced = new CallOrderPlaced();
        BeanUtils.copyProperties(this, callorderPlaced);
        callorderPlaced.publishAfterCommit();
        System.out.println("\n\n##### CallOrderService : onPostPersist()" + "\n\n");
        System.out.println("\n\n##### callorderplace : "+callorderPlaced.toJson() + "\n\n");
        System.out.println("\n\n##### productid : "+this.productId + "\n\n");
        logger.debug("CallOrderService");
    }

    @PostUpdate
    public void onPostUpdate() {
    	
    	CallOrderCanceled callorderCanceled = new CallOrderCanceled();
        BeanUtils.copyProperties(this, callorderCanceled);
        callorderCanceled.publishAfterCommit();
    }
    
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getUsername() {
        return username;
    }
    
....생략 

```

Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 하였고 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

CallOrderRepository.java

```
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel = "callorders", path = "callorders")
public interface CallOrderRepository extends PagingAndSortingRepository<CallOrder, Long> {
	
}
```

- 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
가능한 현업에서 사용하는 언어를 모델링 및 구현시 그대로 사용하려고 노력하였다. 

- 적용 후 Rest API의 테스트
전화주문 결제 후 productdelivery 주문 접수하기 POST

```
[시나리오1]
http POST http://localhost:8085/callorders address=“England” productId=“2001” payStatus=“Y” phoneNo=“01075722744” productName=“gram” productPrice=9000000 qty=1 userId=“gentleman” username=“TRUMP”

[시나리오2] 
http GET http://localhost:8081/stockDeliveries
http GET http://localhost:8082/orderStatus
http GET http://localhost:8083/promotes
http GET http://localhost:8085/callorders
```



- callorder GET 

![get_callorder](https://user-images.githubusercontent.com/88864433/135461817-59ce4da2-914d-41d7-9b0e-b61504ab7661.PNG)

- 기존기능 중 하나인 promotes 

![get_promotes](https://user-images.githubusercontent.com/88864433/135461867-8ae193b3-5a90-499c-b1c0-5bf96bbb43ff.PNG)



# 비동기식 호출과 Eventual Consistency

(이벤트 드리븐 아키텍처)

- 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?

전화주문/전화주문취소 후에 이를 배송팀에 알려주는 트랜잭션은 Pub/Sub 관계로 구현하였다.
아래는 주문/주문취소 이벤트를 통해 kafka를 통해 배송팀 서비스에 연계받는 코드 내용이다. 

```

    @PostPersist
    public void onPostPersist(){
    	
         Logger logger = LoggerFactory.getLogger(this.getClass());

    	
        CallOrderPlaced callorderPlaced = new CallOrderPlaced();
        BeanUtils.copyProperties(this, callorderPlaced);
        callorderPlaced.publishAfterCommit();
        System.out.println("\n\n##### CallOrderService : onPostPersist()" + "\n\n");
        System.out.println("\n\n##### callorderplace : "+callorderPlaced.toJson() + "\n\n");
        System.out.println("\n\n##### productid : "+this.productId + "\n\n");
        logger.debug("CallOrderService");
    }

    @PostUpdate
    public void onPostUpdate() {
    	
    	CallOrderCanceled callorderCanceled = new CallOrderCanceled();
        BeanUtils.copyProperties(this, callorderCanceled);
        callorderCanceled.publishAfterCommit();
    }
```

- 배송팀에서는 전화주문/전화주문취소 접수 이벤트에 대해 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler를 구현한다. 

```
Service
public class PolicyHandler{
    @Autowired StockDeliveryRepository stockDeliveryRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderPlaced_AcceptOrder(@Payload OrderPlaced orderPlaced){

        if(!orderPlaced.validate()) return;

        // delivery 객체 생성 //
         StockDelivery delivery = new StockDelivery();

         delivery.setOrderId(orderPlaced.getId());
         delivery.setUserId(orderPlaced.getUserId());
         delivery.setOrderDate(orderPlaced.getOrderDate());
         delivery.setPhoneNo(orderPlaced.getPhoneNo());
         delivery.setProductId(orderPlaced.getProductId());
         delivery.setQty(orderPlaced.getQty()); 
         delivery.setDeliveryStatus("delivery Started");

         System.out.println("==================================");
         System.out.println(orderPlaced.getId());
         System.out.println(orderPlaced.toJson());
         System.out.println("==================================");
         System.out.println(delivery.getOrderId());

         stockDeliveryRepository.save(delivery);

    }
    
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderCanceled_CancleOrder(@Payload OrderCanceled orderCanceled) {
    	
    	if(!orderCanceled.validate()) return;
... 중략
        for (StockDelivery delivery:deliveryList)
        {
        	System.out.println("\n\n"+orderCanceled.getId());
            delivery.setDeliveryStatus("delivery Canceled");
            stockDeliveryRepository.save(delivery);
        }
     
    }

}
```
- Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가?

오더(order) 서비스의 포트를 추가 (오더 : 8082, 전화오더 : 8085) 하여 2개의 노드로 배송서비스를 실행한다. 


# API 게이트웨이
- API GW를 통하여 마이크로 서비스들의 진입점을 통일할 수 있는가?

- application.yml
```
![api게이트웨이](https://user-images.githubusercontent.com/88864433/135473378-c8f49a7f-3ac9-4ea1-b1db-aa302f8b1789.PNG)

```

Gateway의 application.yml이며, 마이크로서비스들의 진입점을 세팅하여 URI Path에 따라서 각 마이크로서비스로 라우팅되도록 설정되었다.
Gateway 포트인 8085 (callorder)포트를 통해서 주문접수를 생성시켜 8081(배송팀) 에서 정상 동작함을 확인하였다. 

- callorder POST 

![post_callorder](https://user-images.githubusercontent.com/88864433/135461482-8bd3b542-28ea-4d98-8d95-3208f7fd2dc9.PNG)

- stockdelivery GET 

![get_stockdelivery](https://user-images.githubusercontent.com/88864433/135474090-af36b19f-c33d-4a31-8ff6-5a8efa57a905.PNG)


