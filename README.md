# 조별과제 - 12번가

![12번가](https://user-images.githubusercontent.com/88864433/133467597-709524b1-4613-4dab-bc57-948f433ad565.png)
---------------------------------

# Table of contents

- [조별과제 - 12번가 배송서비스](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [DDD 의 적용](#DDD의-적용)
    - [동기식 호출과 Fallback 처리](#동기식-호출과-Fallback-처리) 
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency) 
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [API 게이트웨이](#API-게이트웨이)
  - [운영](#운영)
    - [Deploy/Pipeline](#deploypipeline)
    - [동기식 호출 / Circuit Breaker / 장애격리](#동기식-호출-circuit-breaker-장애격리)
    - [Autoscale (HPA)](#autoscale-(hpa))
    - [Zero-downtime deploy (Readiness Probe)](#zerodowntime-deploy-(readiness-Probe))
    - [Self-healing (Liveness Probe)](#self-healing-(liveness-probe))




# 서비스 시나리오
	
[서비스 아키텍쳐]
주문팀, 상품배송팀, 마케팅팀

[서비스 시나리오]

가. 기능적 요구사항

 1. [주문팀]고객이 상품을 선택하여 주문 및 결제 한다.

 2. [주문팀]주문이 되면 주문 내역이 상품배송팀에게 전달된다. orderplaced

 3. [상품배송팀]상품팀은 재고를 확인해서 배송을 출발한다.

 4. [마케팅팀]배송이 시작되면 쿠폰을 발행한다. ( 4.Req/Resp )

 5. [주문팀]고객이 주문을 취소한다.

 6. [상품배송팀] 주문이 취소되면 배달이 취소된다.


나. 비기능적 요구사항

1. [설계/구현]Req/Resp : 쿠폰이 발행된 건에 한하여 배송을 시작한다. 

2. [설계/구현]CQRS : 고객이 주문상태를 확인 가능해야한다.

3. [설계/구현]Correlation : 주문을 취소하면 배달을 취소->재고 증가 -> 주문 상태 변경

4. [설계/구현]saga : 상품배송팀 기능이 수행되지 않아도 주문은 항상 받을 수 있어야 한다.

5. [설계/구현/운영]circuit breaker : 주문결재건수가 많으면(3건) 주문 및 결재를 받지 않고 잠시 후에 하도록 안내한다.

다. 기타 

1. [설계/구현/운영]polyglot : 주문팀과 상품배송팀은 mysql 데이터베이스 활용



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

![1](https://user-images.githubusercontent.com/88864433/133356420-db8f0cf8-a3f6-4d24-8242-e9e739401045.PNG)
 
### 부적격 이벤트 탈락

![2](https://user-images.githubusercontent.com/88864433/133356470-ee9c68e5-50c7-45b8-8bf2-15b9ee408036.PNG)

```
- 과정 중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
- ‘재고가 충족됨’, ‘재고가 부족함’ 은 배송 이벤트를 수행하기 위한 내용에 가까우므로 이벤트에서 제외
- 주문과 결제는 동시에 이루어진다고 봐서 주문과 결제를 묶음 
```

![3](https://user-images.githubusercontent.com/88864433/133356499-0fa6c5d6-b0ae-48a5-8e9c-06a5bac07ea1.PNG)

### 액터, 커맨드를 부착하여 읽기 좋게 

![4](https://user-images.githubusercontent.com/88864433/133261133-569d5f5d-5411-49fe-a439-30352ed2b0f6.PNG)

 
### 어그리게잇으로 묶기

![5](https://user-images.githubusercontent.com/88864433/133261206-85a09dd0-b646-4e4a-b499-8009f92570a1.PNG)
 
``` 
- 고객의 주문, 점주의 배송관리, 마케팅의 쿠폰관리는 command와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어줌
```

### 바운디드 컨텍스트로 묶기

![6](https://user-images.githubusercontent.com/88864433/133261300-7763fea1-0a66-414d-9c5e-99f6707c1524.PNG)
 
```
- 도메인 서열 분리 
    - Core Domain:  order, delivery : 없어서는 안될 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 order의 경우 1주일 1회 미만, delivery의 경우 1개월 1회 미만
    - Supporting Domain:  marketing : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함
```
### 폴리시 부착


![폴리시](https://user-images.githubusercontent.com/88864433/133261468-49081d3c-a46d-4c5c-a8d5-4f3f8345fde2.PNG)
 

### 폴리시의 이동과 컨텍스트 맵핑 (점선은 Pub/Sub, 실선은 Req/Resp) 

![8](https://user-images.githubusercontent.com/88864433/133361234-7bde60ba-2b5e-415e-a417-924e62d712a4.PNG)
 

### 완성된 모형

![모델](https://user-images.githubusercontent.com/88864433/133361343-d99b4182-22ac-4881-aeee-19ae121723b5.PNG)
 
### 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![주문완료검증](https://user-images.githubusercontent.com/88864433/133361542-bc0225f1-d540-42d8-ab1b-f9de9967e84a.PNG)

```
- 고객이 물건을 주문하고 결제한다 (ok)
- 결제가 완료되면 주문 내역이 배송팀에 전달된다 (ok)
- 마케팅팀에서 쿠폰을 발행한다 (ok) 
- 쿠폰이 발행된 것을 확인하고 배송을 시작한다 (ok)
```
![주문취소검증](https://user-images.githubusercontent.com/88864433/133361562-11bef187-a52e-4948-a429-995d76d4424d.PNG)

``` 
- 고객이 주문을 취소할 수 있다 (ok)
- 주문을 취소하면 결제도 함께 취소된다 (ok)
- 주문이 취소되면 배송팀에 전달된다 (ok)
- 마케팅팀에서 쿠폰발행을 취소한다 (ok)
- 쿠폰발행이 취소되면 배송팀에서 배송을 취소한다 (ok)
```

### 비기능 요구사항에 대한 검증 (5개가 맞는지 검토 필요)

![비기능적 요구사항](https://user-images.githubusercontent.com/88864433/133370337-84871067-789e-48d1-89e4-486fd6d3e00e.PNG)

```
1. [설계/구현]Req/Resp : 쿠폰이 발행된 건에 한하여 배송을 시작한다. 
2. [설계/구현]CQRS : 고객이 주문상태를 확인 가능해야한다.
3. [설계/구현]Correlation : 주문을 취소하면 배달을 취소->재고 증가 -> 주문 상태 변경
4. [설계/구현]saga : 상품배송팀 기능이 수행되지 않아도 주문은 항상 받을 수 있어야 한다.
5. [설계/구현/운영]circuit breaker : 주문결재건수가 많으면(3건) 주문 및 결재를 받지 않고 잠시 후에 하도록 안내한다.
``` 

### 헥사고날 아키텍처 다이어그램 도출 (그림 수정필요없는지 확인 필요)

![분산이벤트스트림](https://user-images.githubusercontent.com/88864433/133448231-162975ce-3bf4-412a-8de6-e419df515834.PNG)
 

```
- Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
- 호출관계에서 PubSub 과 Req/Resp 를 구분함
- 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
```

# 구현
--
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 바운더리 컨텍스트 별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd order
mvn spring-boot:run

cd productdelivery 
mvn spring-boot:run

cd marketing
mvn spring-boot:run 
```

# DDD의 적용 (작성완료) 
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가? 

각 서비스 내에 도출된 핵심 Aggregate Root 객체를 Entity로 선언하였다. (주문(order), 배송(productdelivery), 마케팅(marketing)) 

주문 Entity (Order.java) 
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

    	
        OrderPlaced orderPlaced = new OrderPlaced();
        BeanUtils.copyProperties(this, orderPlaced);
        orderPlaced.publishAfterCommit();
        System.out.println("\n\n##### OrderService : onPostPersist()" + "\n\n");
        System.out.println("\n\n##### orderplace : "+orderPlaced.toJson() + "\n\n");
        System.out.println("\n\n##### productid : "+this.productId + "\n\n");
        logger.debug("OrderService");
    }

    @PostUpdate
    public void onPostUpdate() {
    	
    	OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        orderCanceled.publishAfterCommit();
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

    public void setUsername(String username) {
        this.username = username;
    }
    
....생략 

```

Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 하였고 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

#### [주석] java 소스 구현한 방법에 대한 간략한 설명 필요

promote.java 

``` 
@Entity
@Table(name="Promote_table")
public class Promote {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String phoneNo;
    private String username;
    private Long orderId;
    private String orderStatus;
    private String productId;
    private String payStatus;
    private String couponId;
    private String couponKind;
    private String couponUseYn;
    private String userId;

    @PostPersist
    public void onPostPersist(){
        CouponPublished couponPublished = new CouponPublished();
        BeanUtils.copyProperties(this, couponPublished);
        couponPublished.publishAfterCommit();

    }

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

	public String getPhoneNo() {
		return phoneNo;
	}
.... 생략 

```

- 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
가능한 현업에서 사용하는 언어를 모델링 및 구현시 그대로 사용하려고 노력하였다. 

- 적용 후 Rest API의 테스트
주문 결제 후 productdelivery 주문 접수하기 POST

```
#### (명령어수정필요)
http localhost:8082/ordermgmts orderId=1 itemId=1 itemName="ITbook" qty=1 customerName="HanYongSun" deliveryAddress="kyungkido sungnamsi" deliveryPhoneNumber="01012341234" orderStatus="order"
#### POST 캡쳐화면 

```
#### 작성필요
order 주문 취소하기 PATCH
```
http PATCH localhost:8088/orders/5 orderStatus="orderCanceled"
```

#### 주문취소하기 캡쳐화면


# 동기식 호출과 Fallback 처리 (일부작성완료)

(Request-Response 방식의 서비스 중심 아키텍처 구현)

- 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)

요구사항대로 배송팀에서는 쿠폰이 발행된 것을 확인한 후에 배송을 시작한다.

Promote.java Entity Class에 @PostPersist로 주문 생성 직후 결제를 호출하도록 처리하였다

```
   @PostPersist
    public void onPostPersist(){
        CouponPublished couponPublished = new CouponPublished();
        BeanUtils.copyProperties(this, couponPublished);
        couponPublished.publishAfterCommit();
    }
    
```
... 이후에 작성해야 함. 

# 비동기식 호출과 Eventual Consistency 
-- 주문취소 후에 배송이 취소되는 과정이 비동기식일까?? 

(이벤트 드리븐 아키텍처)

- 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
- Correlation-key: 각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?



#### 답변
...작성필요 
예시) 
카프카를 이용하여 주문완료 시 결제 처리를 제외한 나머지 모든 마이크로서비스 트랜잭션은 Pub/Sub 관계로 구현하였다.
아래는 주문취소 이벤트(OrderCanceled)를 카프카를 통해 주문관리(ordermanagement) 서비스에 연계받는 코드 내용이다.


# SAGA 패턴 
- 취소에 따른 보상 트랜잭션을 설계하였는가(Saga Pattern)

#### 답변 : 
상품배송팀의 기능을 수행할 수 없더라도 주문은 항상 받을 수 있게끔 설계하였다. 

### SAGA 패턴에 맞춘 트랜잭션 실행 (캡쳐화면) 




# CQRS (작성완료. 검토필요) 
- CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

주문/배송상태가 바뀔 때마다 고객이 현재 상태를 확인할 수 있어야 한다는 요구사항에 따라 주문 서비스 내에 OrderStatus View를 모델링하였다

OrderStatus.java 
```
@Entity
@Table(name="OrderStatus_table")
public class OrderStatus {

        @Id
        @GeneratedValue(strategy=GenerationType.AUTO)
        private Long id;
        private String username;
        private String userId;
        private Long orderId;
        private String orderStatus;
        private String productId;
        private String productName;
        private Long productPrice;
        private int qty; 
        private String couponId;
        private String couponKind;
        private String couponUseYn;
.... 생략 
```

OrderStatusViewHandler 를 통해 구현

Pub/Sub 기반으로 별도 ProductPage_table 테이블에 저장되도록 구현하였다.

```
@Service
public class OrderStatusViewHandler {


    @Autowired
    private OrderStatusRepository orderStatusRepository;
    
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderPlaced_then_CREATE_1 (@Payload OrderPlaced orderPlaced) {
        try {

            if (!orderPlaced.validate()) return;

            // view 객체 생성
            OrderStatus orderStatus = new OrderStatus();
            orderStatus.setUsername(orderPlaced.getUsername());
            orderStatus.setUserId(orderPlaced.getUserId());
            orderStatus.setOrderId(orderPlaced.getId());
            orderStatus.setOrderStatus("OrderPlaced");
            orderStatus.setProductId(orderPlaced.getProductId());
            orderStatus.setProductName(orderPlaced.getProductName());
            orderStatus.setProductPrice(orderPlaced.getProductPrice());
            orderStatus.setQty(orderPlaced.getQty());
           
            orderStatusRepository.save(orderStatus);
            
            System.out.println("\n\n##### OrderStatus : whenOrderPlaced_then_CREATE_1" + "\n\n");

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

주문에 대한 결제완료(PayStatus) 시 orderId를 키값으로 OrderStatus 데이터도 생성되며 (주문과 결제를 동시에 처리했을 때 배송을 시작하므로)

"결제완료(주문완료), 주문접수, 배송시작, 결제취소(주문취소)"의 이벤트에 따라 주문상태가 업데이트되도록 모델링하였다.




- CQRS 테스트 
``` 
캡쳐화면 등록 
````

- Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
#### 답변 
ordermanagement 서비스만 구동되고 delivery 서비스는 멈춰있는 상태이다. 주문관리에 이벤트가 발생하면 카프카 큐에 정상적으로 들어감을 확인할 수 있다.

# 폴리글랏 퍼시스턴스 (작성완료. 검토필요) 
- pom.xml
```
		<dependency>
        	<groupId>mysql</groupId>
        	<artifactId>mysql-connector-java</artifactId>
        	<scope>provided</scope>
    	</dependency>

		<dependency>
		    <groupId>org.javassist</groupId>
    		<artifactId>javassist</artifactId>
    		<version>3.25.0-GA</version>
		</dependency>
```

application.yml
```

spring:
  profiles: docker

  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://cloud12st.ck7n6wloicx4.ap-northeast-2.rds.amazonaws.com:3306/cloud12st
    username: root
    password: cloud#1234

  jpa:
    open-in-view: false
    show-sql: true
    hibernate:
      format_sql: true
      ddl-auto: create
```

- 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?

#### 답변 
H2 DB의 경우 휘발성 데이터의 단점이 있는데, productdelivery 서비스의 경우 타 서비스들의 비해 중요하다고 생각하였다.
productdelivery는 주문과 쿠폰발행/취소를 중간에서 모두 파악하여 처리해야 되기 때문에 백업,복원기능과 안정성이 장점이 있는 mysql을 선택하여 구현하였다.


# API 게이트웨이 
--
- API GW를 통하여 마이크로 서비스들의 진입점을 통일할 수 있는가?
#### 답변
아래는 MSAEZ를 통해 자동 생성된 gateway 서비스의 application.yml이며, 마이크로서비스들의 진입점을 통일하여 URL Path에 따라서 마이크로서비스별 서로 다른 포트로 라우팅시키도록 설정되었다.

# 운영
--
# Deploy/Pipeline (작성완료)

- (CI/CD 설정) BuildSpec.yml 사용 각 MSA 구현물은 git의 source repository 에 구성되었고, AWS의 CodeBuild를 활용하여 무정지 CI/CD를 설정하였다.

- Repository 화면 캡쳐 

![CICD](https://user-images.githubusercontent.com/88864433/133468925-a9ba1fec-8331-4a68-a0b7-2b570e4182de.PNG)

- CodeBuild 설정

![CODEBUILD1](https://user-images.githubusercontent.com/88864433/133469657-2b250c1e-777d-4d18-8ae9-c631ba9fa9f6.PNG)


![codebuild2](https://user-images.githubusercontent.com/88864433/133469760-d091efc6-5d09-4c25-a324-337f0b5e0d87.PNG)

- 빌드 환경 설정 
환경변수(KUBE_URL, KUBE_TOKEN, repository 등 설정) 

![codebuild_환경변수](https://user-images.githubusercontent.com/88864433/133470474-c69371cd-2ed6-49f1-adb5-8d1f7ac4d056.PNG)


- buildspec.yml

```
version: 0.2
​
env:
  variables:
    IMAGE_REPO_NAME: "order"
    CODEBUILD_RESOLVED_SOURCE_VERSION: "latest"
​
phases:
  install:
    commands:    
      - nohup /usr/local/bin/dockerd --host=unix:///var/run/docker.sock --host=tcp://127.0.0.1:2375 --storage-driver=overlay2&
      - timeout 15 sh -c "until docker info; do echo .; sleep 1; done"
    runtime-versions:
      java: corretto11
      docker: 18
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - echo $IMAGE_REPO_NAME
      - echo $AWS_ACCOUNT_ID
      - echo $AWS_DEFAULT_REGION
      - echo $CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo start command
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION  .
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
​
cache:
  paths:
    - '/root/.m2/**/*' 
```

# 동기식 호출 / Circuit Breaker / 장애격리
서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함
오더 요청이 과도할 경우 서킷 브레이크를 통해 장애 격리를 하려고 한다.

Hystrix 를 설정: 요청처리 쓰레드에서 처리시간이 610 ms가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정

# Autoscale(HPA) 
-- 

# Zero-downtime deploy (Readiness Probe) 
-- 
(무정지 배포) 서비스의 무정지 배포를 위하여 주문관리(Ordermanagement) 서비스의 배포 yaml 파일에 readinessProbe 옵션을 추가하였다.



# Self-healing (Liveness Probe) 
-- 
주문관리(Ordermanagement) 서비스의 배포 yaml 파일에 Pod 내 /tmp/healthy 파일을 5초마다 체크하도록 livenessProbe 옵션을 추가하였다
