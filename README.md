![저기어때](https://user-images.githubusercontent.com/73212897/96727589-2d6a0880-13ee-11eb-91ab-acf6e0cc9526.jpg)


# 객실예약서비스
- 4조

# Table of contents

- [객실예약서비스](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

기능적 요구사항
1. 숙박관리자는 객실을 등록할 수 있다
2. 고객이 객실을 선택하여 예약한다.
3. 예약이 완료되면 해당 객실은 '예약불가' 상태로 변경된다.
4. 결제가 완료되면 객실 예약이 확정된다.
5. 고객이 예약을 취소할 수 있다.
6. 숙박관리자가 객실을 체크아웃 하면 객실은 예약가능 상태로 변경된다.
7. 고객이 숙소 예약상태를 중간중간 조회한다.
8. 예약상태가 바뀔 때 마다 카톡으로 알림을 보낸다

비기능적 요구사항
1. 트랜잭션

    i. 결제가 되지 않은 예약건은 아예 예약이 완료되지 않아야 한다  Sync 호출 
2. 장애격리

    i. 객실관리시스템이 수행되지 않더라도 객실 예약은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    
    ii. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
3. 성능

    i. 고객이 객실상태를 예약시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS    
    
    ii. 객실상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다 Event driven


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
      - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
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
    - 서킷브레이커를 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?

  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등

# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
![image](https://user-images.githubusercontent.com/69283674/97150539-99af8800-17b1-11eb-8b8e-c6b8bcd1d537.png)

## TO-BE 조직 (Vertically-Aligned)
![image](https://user-images.githubusercontent.com/69283674/97150593-b350cf80-17b1-11eb-8c7f-68c5be80badc.png)

[조직 KPI]
예약팀 : 고객의 예약을 최대한 많이 받아야 함.
결제팀 : 결제 안정성을 최대한 확보해야 함.
객실팀 : 고객의 객실평가 점수가 높아야 함.

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: 
https://www.msaez.io/#/storming/vK3Ti7jb85Q5GVnPwKO5ecQpjRJ2/every/a1a546e3387be89639c5a1c96210c4bb/-MKbLyDZELkpG9pKR7Qa



### 이벤트 도출
![image](https://user-images.githubusercontent.com/69283674/97150926-307c4480-17b2-11eb-8c70-251530e0d9fd.png)
    
### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/69283674/97151026-4ee24000-17b2-11eb-8b90-2e035743d335.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
        - 결재시>결재버튼이 클릭됨  : UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외

### 액터 및 커맨드 부착하여 읽기 좋게

![image](https://user-images.githubusercontent.com/69283674/97151489-e8a9ed00-17b2-11eb-879d-75de8f6a129b.png)


### 어그리게잇으로 묶기

![image](https://user-images.githubusercontent.com/69283674/97152033-af25b180-17b3-11eb-91dd-cfe3e514cfd6.png)

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/69283674/97152594-79cd9380-17b4-11eb-9bbe-5e0f02421917.png)

    - 도메인 서열 분리 
        - Core Domain:  Reservation : 없어서는 안될 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만
        - Supporting Domain:   Room : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   Payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![image](https://user-images.githubusercontent.com/69283674/97152713-a4b7e780-17b4-11eb-9bb5-8f498bf5de5f.png)


### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/69283674/97153018-2871d400-17b5-11eb-9773-276a38813eb9.png)


### 1차 완성본

![image](https://user-images.githubusercontent.com/69283674/97153154-63740780-17b5-11eb-904e-74bb90a97738.png)

### 기능적 요구사항을 커버하는지 검증(1)

![image](https://user-images.githubusercontent.com/69283674/97153235-81416c80-17b5-11eb-8d93-ab678184e2c7.png)

기능적 요구사항

    - 숙박관리자는 객실을 등록할 수 있다 (OK)
   
    - 고객이 객실을 선택하여 예약한다 (OK)
    
    - 예약이 완료되면 해당 객실은 '예약불가' 상태로 변경된다 (OK)
    
    - 결제가 완료되면 예약이 확정된다 (OK)

### 기능적 요구사항을 커버하는지 검증(2)

![image](https://user-images.githubusercontent.com/69283674/97153635-28260880-17b6-11eb-9e2c-225e1407d31f.png)

기능적 요구사항

    - 고객은 예약을 취소할 수 있다 (OK)
    
    - 예약이 취소되면 해당 객실은 '예약가능' 상태로 변경된다 (OK)
    
    - 고객이 숙소 예약상태를 중간중간 조회한다 (OK)
    
    - 예약상태가 바뀔떄마다 알람을 보낸다 (OK)


### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/69283674/97153970-ada9b880-17b6-11eb-877b-95b4f9813a7d.png)

1. 트랜잭션
 - 결제가 되지 않은 예약건은 아예 예약이 완료되지 않아야 한다 Sync 호출
 -> 예약 시 결제처리:  결제가 완료되지 않은 주문은 절대 받지 않는다는 경영자의 오랜 신념(?) 에 따라, ACID 트랜잭션 적용. 예약완료시 결제처리에 대해서는 Request-Response 방식 처리
 
2. 장애격리
 - 객실관리시스템이 수행되지 않더라도 객실 예약은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
 -> 결제 완료시 객실연결 및 확정처리:  App(front) 에서 Room 마이크로서비스로 예약요청이 전달되는 과정에 있어서 Room 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
 - 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
 
3. 성능
 - 고객이 객실상태를 수신하여 예약시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS
 - 예약상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다 Event driven
 -> 나머지 모든 inter-microservice 트랜잭션: 예약상태, 결제상태 등 모든 이벤트에 대해 카톡을 처리하는 등, 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.
 
## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/69283674/97154475-40e2ee00-17b7-11eb-9bd6-1d19a6f9e157.png)

    - Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    
    

# 변화관리:

## 사업 확대 및 고객 만족도 향상을 위한 프로모션 진행 요구사항 추가 (프로모션 상품 배송)

## 기능적 요구사항
1. 고객 예약 주문시, 프로모션 상품을 배송 한다.
2. 프로모션 배송 완료 되면, 관리자가 예약 시스템을 통하여 배송완료 처리한다.
3. 프로모션 상품의 배송 완료시, 고객에게 알림을 발송한다.

## 비기능적 요구사항
1. (트랜잭션) 프로모션 상품은 회원 약관에 반영되어 있으므로, 예약 주문 접수시 반드시 배송되어야 한다.






# 구현:

요구사항의 추가에 따라 신규 서버스(마이크로 서비스)를 추가하고, 스프링부트로 구현하였다. 
서비스의 추가 이후 전체 서비스는 아래와 같으며, 로컬에서 실행하는 방법을 기재한다. (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd Reservation
mvn spring-boot:run

cd Payment
mvn spring-boot:run 

cd Room
mvn spring-boot:run  

cd Notice
mvn spring-boot:run  

cd Gateway
mvn spring-boot:run  

cd Delivery
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

```
package accommodation;

import javax.persistence.*;

import accommodation.external.Payment;
import accommodation.external.PaymentManagementService;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import accommodation.config.kafka.KafkaProcessor;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.util.MimeTypeUtils;

@Entity
@Table(name="Reservation_table")
public class Reservation {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Integer reservationNumber;
    private Integer roomNumber;
    private Integer customerId;
    private String customerName;
    private String reserveStatus;
    private Integer PaymentPrice;

    public Integer getReservationNumber() {
        return reservationNumber;
    }

    public void setReservationNumber(Integer reservationNumber) {
        this.reservationNumber = reservationNumber;
    }
    public String getCustomerName() {
        return customerName;
    }

    public void setCustomerName(String customerName) {
        this.customerName = customerName;
    }
    public Integer getCustomerId() {
        return customerId;
    }

    public void setCustomerId(Integer customerId) {
        this.customerId = customerId;
    }
    public String getReserveStatus() {
        return reserveStatus;
    }

    public void setReserveStatus(String reserveStatus) {
        this.reserveStatus = reserveStatus;
    }
    public Integer getRoomNumber() {
        return roomNumber;
    }

    public void setRoomNumber(Integer roomNumber) {
        this.roomNumber = roomNumber;
    }

    public Integer getPaymentPrice() {
        return PaymentPrice;
    }

    public void setPaymentPrice(Integer paymentPrice) {
        PaymentPrice = paymentPrice;
    }


}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package accommodation;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface ReservationRepository extends PagingAndSortingRepository<Reservation, Integer>{

}
```
- 적용 후 REST API 의 테스트

```
#  Reservation 서비스의 예약 주문
http post http://reservation:8080/reservations customerName="kosmocor" customerId=12345 reserveStatus="reserve" roomNumber=1 paymentPrice=50000 deliveryStatus="DeliveryRequested"
```
--> 예약주문 이미지

```
# 예약 상태 확인
http http://reservation:8080/reservations
```
--> 예약상태 이미지

```
# 프로모션 상품 배송 요청 확인
http http://delivery:8080/deliveries
```
--> 배송요청 이미지




## 동기식 호출 

추가된 요구사항으로 예약(Reservation)->배송(Delivery) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
```
# (delivery) DeliveryService.java

package accommodation.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;
...
@org.springframework.stereotype.Service
@FeignClient(name="deliveryNumber", url="${api.url.delivery}")
public interface DeliveryService {

    @RequestMapping(method= RequestMethod.POST, path="/deliveries", consumes = "application/json")
    public void RequestDelivery(@RequestBody Delivery delivery);
}
```

- 접수를 받은 직후(@PrePersist) 배송 요청하도록 처리
```
# (reservation) Reservation.java
...
    @PrePersist
    public void onPrePersist(){
        ...
        if ("DeliveryRequested".equals(deliveryStatus) ) {
            Delivery delivery = new Delivery();

            delivery.setDeliveryStatus(this.getDeliveryStatus());
            delivery.setReservationNumber(this.getReservationNumber());
            delivery.setCustomerId(this.getCustomerId());
            delivery.setCustomerName(this.getCustomerName());

            Application.applicationContext.getBean(DeliveryService.class).RequestDelivery(delivery);
            ...
        }
    }
```

- 동기식 호출에서는 연동된 결제 시스템이 장애가 나면, 커플링되어 객실 예약이 안되는 현상 확인함.
```
# 배송(delivery) 서비스 down

# 객실(reservation) 예약 실패
http post http://reservation:8080/reservations reservationNumber=1 reserveStatus="payment" customerName="PARK" customerId=9805 roomNumber=1 paymentPrice=50001
```
  -> 객실 예약 실패 이미지

```
# 배송(delivery) 서비스 재기동
cd delivery
mvn spring-boot:run

# 객실(reservation) 예약 성공
```
  -> 객실 예약 실패 이미지



## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

예약주문/배송요청 이후 배송완료의 처리는 동기식이 아니라 비 동기식으로 처리하여, 배송 시스템의 일시 중단이 예약/결재/배송완료의 처리가 블로킹 되지 않도록 적용하였다.
- 이를 위해 예약 정보의 수정 이후 곧바로 배송 완료 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
```
# (reservation) Reservation.java
...
    @PreUpdate
    public void onPreUpdate(){
        ....
        if ("DeliveryCompleted".equals(deliveryStatus) ) {
            DeliveryCompleted deliveryCompleted = new DeliveryCompleted();

            deliveryCompleted.setDeliveryStatus(deliveryStatus);
            deliveryCompleted.setReservationNumber(reservationNumber);
            deliveryCompleted.setCustomerId(customerId);
            deliveryCompleted.setCustomerName(customerName);

            ObjectMapper objectMapper = new ObjectMapper();
            String json = null;

            try {
                json = objectMapper.writeValueAsString(deliveryCompleted);
            } catch (JsonProcessingException e) {
                throw new RuntimeException("JSON format exception", e);
            }

            KafkaProcessor processor = Application.applicationContext.getBean(KafkaProcessor.class);
            MessageChannel outputChannel = processor.outboundTopic();

            outputChannel.send(MessageBuilder
                    .withPayload(json)
                    .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                    .build());
            System.out.println(deliveryCompleted.toJson());
            deliveryCompleted.publishAfterCommit();
        }
        ...
    }

```

배송(Delivery) 시스템에서는 배송완료 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현하였다.
 
```
@Service
public class PolicyHandler {
    @Autowired
    DeliveryRepository deliveryRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverDeliveryCompleted(@Payload DeliveryCompleted deliveryCompleted){
        // 배송 완료시, 배송상태 Update
        if(deliveryCompleted.isMe()){
            Delivery delivery = new Delivery();
            delivery.setDeliveryStatus(deliveryCompleted.getDeliveryStatus());
            delivery.setReservationNumber(deliveryCompleted.getReservationNumber());
            delivery.setDeliveryStatus(deliveryCompleted.getDeliveryStatus());
            delivery.setCustomerId(deliveryCompleted.getCustomerId());
            delivery.setCustomerName(deliveryCompleted.getCustomerName());
            delivery.setCustomerAddress(deliveryCompleted.getCustomerAddress());
            delivery.setCustomerContract(deliveryCompleted.getCustomerContract());

            deliveryRepository.save(delivery);
        }
    }
}
```



## CQRS 패턴 
사용자 View를 위한 객실 정보 조회 서비스를 위한 별도의 객실 정보 저장소를 구현
- 이를 하여 RoomInfo 서비스를 별도로 구축하고 저장 이력을 기록한다.
- 모든 정보는 비동기 방식으로 호출한다.

```
Room.java(Entity)

@Entity
@Table(name="Room_table")
public class Room {
    ...

    @PostPersist
    public void onPostPersist(){
        RoomConditionChanged roomConditionChanged = new RoomConditionChanged();
        roomConditionChanged.setRoomNo(this.getRoomNo());
        roomConditionChanged.setRoomType(this.getRoomType());
        roomConditionChanged.setRoomStatus(this.getRoomStatus());
        roomConditionChanged.setRoomName(this.getRoomName());
        roomConditionChanged.setRoomQty(this.getRoomQty());
        roomConditionChanged.setRoomPrice(this.getRoomPrice());

        BeanUtils.copyProperties(this, roomConditionChanged);

        roomConditionChanged.publishAfterCommit();
    }
    @PostUpdate
    public void onPostUpdate(){
        System.out.println("예약가능?:"+this.roomStatus);
        RoomConditionChanged roomConditionChanged = new RoomConditionChanged();
        roomConditionChanged.setRoomNo(this.getRoomNo());
        roomConditionChanged.setRoomType(this.getRoomType());
        roomConditionChanged.setRoomStatus(this.getRoomStatus());
        roomConditionChanged.setRoomName(this.getRoomName());
        roomConditionChanged.setRoomQty(this.getRoomQty());
        roomConditionChanged.setRoomPrice(this.getRoomPrice());

        BeanUtils.copyProperties(this, roomConditionChanged);
        roomConditionChanged.publishAfterCommit();
        System.out.println("예약가능으로 변경");
    }
```

사용자 View를 위한 객실 정보 조회 서비스를 위한 별도의 객실 정보 저장소를 구현
- 이를 하여 RoomInfo 서비스를 별도로 구축하고 저장 이력을 기록한다.
- 모든 정보는 비동기 방식으로 호출한다.
```
PolicyHandler.java

@Service
public class PolicyHandler{

    @Autowired
    RoomInfoRepository roomInfoRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverSave_RoomInfo(@Payload RoomConditionChanged roomConditionChanged){
        if(roomConditionChanged.isMe()){
            System.out.println("##### listener 객실정보저장 : " + roomConditionChanged.toJson());
            RoomInfo roomInfo = new RoomInfo();
            roomInfo.setRoomNumber(roomConditionChanged.getRoomNumber());
            roomInfo.setRoomName(roomConditionChanged.getRoomName());
            roomInfo.setRoomStatus(roomConditionChanged.getRoomStatus());
            roomInfo.setRoomPrice(roomConditionChanged.getRoomPrice());
            roomInfo.setRoomQty(roomConditionChanged.getRoomQty());
            roomInfo.setRoomStatus(roomConditionChanged.getRoomStatus());
            roomInfo.setRoomType(roomConditionChanged.getRoomType());

            roomInfoRepository.save(roomInfo);
        }
    }
}

```

# API 게이트웨이

Cloud 환경에서는 //서비스명:8080 에서 Gateway API가 작동해야하므로, application.yml 파일에 profile별 gateway 설정을 적용하였다.
-  Gateway 설정 파일 

```
(gateway) application.yml
spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: reservation
          uri: http://localhost:8081
          predicates:
            - Path=/reservations/**
        - id: payments
          uri: http://localhost:8082
          predicates:
            - Path=/payments/**
        - id: rooms
          uri: http://localhost:8083
          predicates:
            - Path=/rooms/**
        - id: notices
          uri: http://localhost:8084
          predicates:
            - Path=/notices/**
        - id: roomInfoes
          uri: http://localhost:8085
          predicates:
            - Path=/roomInfoes/**
        - id: deliveries
          uri: http://localhost:8089
          predicates:
            - Path=/deliveries/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true
---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: reservation
          uri: http://reservation:8080
          predicates:
            - Path=/reservations/**
        - id: payments
          uri: http://payments:8080
          predicates:
            - Path=/payments/**
        - id: rooms
          uri: http://room:8080
          predicates:
            - Path=/rooms/**
        - id: notices
          uri: http://notices:8080
          predicates:
            - Path=/notices/**
        - id: roomInfoes
          uri: http://roomInfo:8080
          predicates:
            - Path=/roomInfoes/**
        - id: deliveries
          uri: http://delivery:8080
          predicates:
            - Path=/deliveries/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080
```



# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AZURE를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 azure-pipelines.yml 에 포함되었다.

-  아래와 같이 pod 가 정상적으로 올라간 것을 확인하였다. 

![image](https://user-images.githubusercontent.com/68646938/97491595-b90f0680-19a5-11eb-9e18-97d0111d9da1.PNG)

-  아래와 같이 쿠버네티스에 모두 서비스로 등록된 것을 확인할 수 있다. 

![image](https://user-images.githubusercontent.com/68646938/97491876-1dca6100-19a6-11eb-82b0-fb10cf727017.PNG)



## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 예약(Reservation)-->배송(DElivery) 시의 연결을 RESTful Request/Response 로 연동하여 구현 되어있고, 배송 요청이 과도할 경우 CB를 통하여 장애격리 한다.
- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 600 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

feign:
  hystrix:
    enabled: true

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 600

```

피호출 서비스(배송:Delivery) 의 임의 부하 처리 - 400 밀리 추가하여 0~300 밀리 정도의 지연이 발생하도록 적용
```
# (Delivery) Delivery.java (Entity)

    @PrePersist
    public void onPrePersist(){  //결제이력을 저장한 후 적당한 시간 끌기

        ...
        
            long delayed = (long) (400 + Math.random() * 300);
            try {
                Thread.currentThread().sleep((long) delayed);
                System.out.println("======= finally delayed : " + delayed);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        ....
    }

```
부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 1명, 20초 동안 실시

![image](https://user-images.githubusercontent.com/68646938/97492160-844f7f00-19a6-11eb-8a50-44c450f335a3.PNG)
![image](https://user-images.githubusercontent.com/68646938/97492191-8dd8e700-19a6-11eb-8947-56d6a4682d84.PNG)

운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 50%의 비율로 성공과 실패를 반복하여 신뢰성 있는 시스템으로 판단하기 어렵기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요할 것으로 판단하였다.




### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- 배송서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:

```
kubectl autoscale deploy delivery --min=1 --max=10 --cpu-percent=15
```
  -> 여기에 replica 확장 이미지 

```
Auto Scale-Out 
kubectl apply -f https://k8s.io/examples/application/php-apache.yaml
NOTE : 서비스가 Auto Scaling되기 위해서는 컨테이너 Spec에 Resources : 설정이 있어야 함
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m  
(오토 스케일링 설정, hpa: HorizontalPodAutoscaler )
kubectl autoscale deployment php-apache --cpu-percent=20 --min=1 --max=10
cpu-percent=50 : Pod 들의 요청 대비 평균 CPU 사용율 (여기서는  요청이 200 milli-cores이므로, 모든 Pod의 평균 CPU 사용율이 100 milli-cores(50%)를 넘게되면 HPA 발생)
```


- 워크로드를 걸어준다.
```
siege -c100 -t60S  --content-type "application/json" 'http://reservation:8080/reservations POST {"reserveStatus":"reserve","roomNumber":1,"paymentPrice":50000,"deliveryStatus":"DeliveryRequested"}'

```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy delivery -w
```

- 어느정도 시간이 흐른 후 스케일 아웃이 벌어지는 것을 확인할 수 있다:

  -> deploy scale-out 이미지



### 무정지 재배포

- 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler과 CB 설정을 제거함
- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t60S  --content-type "application/json" 'http://reservation:8080/reservations POST {"reserveStatus":"reserve","roomNumber":1,"paymentPrice":50000,"deliveryStatus":"DeliveryRequested"}'
```

- 새버전으로의 배포 시작
```
kubectl set image ...

(Product 서비스) 
코드 수정시, Update(이미지 재적용) 하기  
kubectl set image deploy product product=283210891307.dkr.ecr.ap-northeast-2.amazonaws.com/product:v2 로  컨테이너 이미지 Update


```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
 -> Update 이미지... 
Transactions:		        2962 hits
Availability:		       80.45 %
Elapsed time:		       29.99 secs
Data transferred:	        0.75 MB
Response time:		        0.01 secs
Transaction rate:	       98.77 trans/sec
Throughput:		        0.02 MB/sec
Concurrency:		       1.46
```

배포기간중 Availability 가 평소 100%에서 60% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정하였다.

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```


- 동일한 시나리오로 재배포 한 후 Availability 확인:

  -> 이미지 현행화... 
![image](https://user-images.githubusercontent.com/69283674/97297629-a1e0f380-1895-11eb-89f3-acc70aa3a30c.png)

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


## Livness구현

Delivery의 depolyment.yml 소스설정
 - http get방식에서 tcp방식으로 변경, 서비스포트 8080이 아닌 8081로 포트 변경하였다.,
 
 - describe 확인
 
 - 원복 후, 정상 확인




## Configmap
컨테이너 이미지로부터 설정 정보를 분리하기 위한 ConfigMaps을 적용/확인하였다.
- 환경변수나 설정값 들을 환경변수로 관리해 Pod가 생성될 때 이 값을 주입한다.






onfigmap.yaml 파일설정



