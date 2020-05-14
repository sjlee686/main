![image](https://user-images.githubusercontent.com/52994285/81627829-ff919880-9439-11ea-8dde-5fe389db2ef1.png)


# 객실예약서비스
- 3조

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
1. 고객은 예약 가능한 객실을 조회할 수 있다
1. 고객이 객실을 선택하여 예약한다.
1. 고객은 예약을 취소할 수 있다.
1. 예약이 완료되면 해당 객실은 '예약불가' 상태로 변경된다.
1. 결제가 완료되면 예약이 완료된다.
1. 고객은 결제를 취소할 수 있다.
1. 고객이 체크아웃 하면 관리자에게 객실 정비 요청이 된다.
1. 고객은 객실에 대한 평가를 등록한다.

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 예약건은 아예 예약이 완료되지 않아야 한다  Sync 호출 
1. 장애격리
    1. 결제시스템이 수행되지 않더라도 객실 예약은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 자주 객실관리시스템에서 확인할 수 있는 객실상태를 예약시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS


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
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/52994285/81630694-3a4aff00-9441-11ea-8826-482c404585f7.png)

[조직 KPI]
예약팀 : 고객의 예약을 최대한 많이 받아야 함.
결제팀 : 결제 안정성을 최대한 확보해야 함.
객실관리팀 : 고객의 객실평가 점수가 높아야 함.

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: 
![image](https://user-images.githubusercontent.com/36434874/81758539-177e2080-94fd-11ea-8f94-889773315098.png)


### 이벤트 도출 및 부적격 이벤트 탈락

![image](https://user-images.githubusercontent.com/52994285/81631466-e4775680-9442-11ea-94db-ca75e5f90eb5.png)




    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
        - 검색시>객실 조회됨  :  UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외

### 액터 및 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/52994285/81630900-b80f0a80-9441-11ea-9908-5dfb5a81cfd7.png)


### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/52994285/81631668-52238280-9443-11ea-8d44-a08a43e8b898.png)

![image](https://user-images.githubusercontent.com/52994285/81632109-6f0c8580-9444-11ea-8c5c-98bc146ad339.png)

    - 도메인 서열 분리 
        - Core Domain:  ReservationManagement : 없어서는 안될 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만, store 의 경우 1개월 1회 미만
        - Supporting Domain:   RoomManagement : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   paymentManagement : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)

### 폴리시 부착 및 폴리시의 이동 후  컨텍스트 매핑 (파란선은 Pub/Sub, 빨간선은 Req/Resp)

![image](https://user-images.githubusercontent.com/52994285/81631853-d0802480-9443-11ea-9e20-2e97cf4f5640.png)



### 완성된 1차 모형

![image](https://user-images.githubusercontent.com/52994285/81632152-8481af80-9444-11ea-8859-6fd41cba4fde.png)


### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증


![image](https://user-images.githubusercontent.com/52994285/81779409-a015b480-952f-11ea-9d4d-69b12228e8eb.png)

기능적 요구사항

    - 고객은 예약 가능한 객실을 조회할 수 있다
    - 고객이 객실을 선택하여 예약한다.
    - 고객은 예약을 취소할 수 있다.
    - 예약이 완료되면 해당 객실은 '예약불가' 상태로 변경된다.
    - 결제가 완료되면 예약이 완료된다.
    - 고객은 결제를 취소할 수 있다.
    - 고객이 체크아웃 하면 관리자에게 객실 정비 요청이 된다.
    - 고객은 객실에 대한 평가를 등록한다.


### 모델 수정

![image](https://user-images.githubusercontent.com/52994285/81639583-52c61400-9457-11ea-9954-a79ecaeba812.png)
    
    - 수정된 모델은 모든 요구사항을 커버함.
    
    


### 비기능 요구사항에 대한 검증



트랜잭션
 - 결제가 되지 않은 예약건은 아예 예약이 완료되지 않아야 한다 Sync 호출
장애격리
 - 결제시스템이 수행되지 않더라도 객실 예약은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
 - 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
성능
 - 고객이 자주 객실관리시스템에서 확인할 수 있는 객실상태를 예약시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS

   - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
   - 예약 시 결제처리:  결제가 완료되지 않은 주문은 절대 받지 않는다는 경영자의 오랜 신념(?) 에 따라, ACID 트랜잭션 적용. 주문와료시 결제처리에 대해서는 Request-Response 방식 처리
   - 결제 완료시 점주연결 및 배송처리:  App(front) 에서 Store 마이크로서비스로 주문요청이 전달되는 과정에 있어서 Store 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
   - 나머지 모든 inter-microservice 트랜잭션: 주문상태, 배달상태 등 모든 이벤트에 대해 카톡을 처리하는 등, 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.




## 헥사고날 아키텍처 다이어그램 도출
    

![image](https://user-images.githubusercontent.com/52994285/81779998-bb34f400-9530-11ea-9d41-5da175322c2f.png)



    - Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd ReservationManagement
mvn spring-boot:run

cd RoomManagement
mvn spring-boot:run 

cd PaymentManagement
mvn spring-boot:run  

cd gateway
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

```
package hotelmanage;

import javax.persistence.*;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import hotelmanage.config.kafka.KafkaProcessor;
import org.springframework.beans.BeanUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.messaging.Processor;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.util.MimeTypeUtils;

import java.util.List;

@Entity
@Table(name="ReservationManagement_table")
public class ReservationManagement {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Integer reservationNumber;
    private String customerName;
    private Integer customerId;
    private String reserveStatus;
    private Integer roomNumber;
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

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package hotelmanage;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface ReservationManagementRepository extends PagingAndSortingRepository<ReservationManagement, Integer>{

}
```
- 적용 후 REST API 의 테스트
```


#  RoomManagement 서비스의 객실정보처리
http post localhost:8083/roomManagements roomStatus="first"

#  ReservationManagement 서비스의 예약처리
http post localhost:8081/reservationManagements  customerName="Lee" customerId=123 reserveStatus="1" roomNumber=1 paymentPrice=50000

# 예약 상태 확인
http post localhost:8084/rommInfos

```



## 동기식 호출 

분석단계에서의 조건 중 하나로 예약(Reservation)->결제(Payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (payment) PaymentManagementService.java

package hotelmanage.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;

@org.springframework.stereotype.Service
@FeignClient(name="reservationNumber", url="${api.url.payment}")
public interface PaymentManagementService {

    @RequestMapping(method= RequestMethod.POST, path="/payments", consumes = "application/json")
    public void CompletePayment(@RequestBody Payment payment);

}
```

- 결제 요청 시 동기적으로 실행
```
# ReservationManagement.java (Entity)

    @PreUpdate
    public void onPreUpdate(){
        ...
        Application.applicationContext.getBean(hotelmanage.external.PaymentManagementService.class).CompletePayment(payment);
    }
```

- 동기식으로 결제 요청시 결제 시스템 장애의 경우 예약 불가 확인 :
```
#결제요청
http post http://ReservationManagement:8080/reservationManagements reservationNumber=1 reserveStatus="reserve" customerName="Lee" customerId=123 roomNumber=1 paymentPrice=50001

```


```
#객실등록
http post localhost:8083/roomManagements roomStatus="first"

#예약요청 (객실이 있을경우)
http post http://ReservationManagement:8080/reservationManagements  customerName="Lee" customerId=123 reserveStatus="1" roomNumber=1 paymentPrice=50000

#결제요청
http post http://ReservationManagement:8080/reservationManagements reservationNumber=1 reserveStatus="reserve" customerName="Lee" customerId=123 roomNumber=1 paymentPrice=50001

#Check Out
http post http://ReservationManagement:8080/reservationManagements reservationNumber=1 reserveStatus="checkOut" customerName="Lee" customerId=123 roomNumber=1 paymentPrice=50001
```


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


결제 이후 이를 예약 정보 변경은 비동기로 호출하여 개별적으로 조회 가능하도록 구현
 
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package hotelmanage;

@Entity
@Table(name="ReservationManagement_table")
public class ReservationManagement {

 ...
    @PrePersist
    public void onPrePersist(){
        setReserveStatus("reserve");
        Reserved reserved = new Reserved();
        reserved.setReservationNumber(this.getReservationNumber());
        reserved.setReserveStatus(this.getReserveStatus());
        reserved.setCustomerName(this.getCustomerName());
        reserved.setCustomerId(this.getCustomerId());
        reserved.setRoomNumber(this.getRoomNumber());
        reserved.setPaymentPrice(this.getPaymentPrice());

        reserved.publishAfterCommit();
    }
```
- 예약 서비스에서 해당 비동기 호출을 수신할 PolicyHandler를 구현

```
package hotelmanage;

...

@Service
public class PolicyHandler{

    @Autowired
    ReservationManagementRepository reservationManagementrepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentCompleted_ChangeResvStatus(@Payload PaymentCompleted paymentcompleted){
        System.out.println(paymentcompleted.toJson());
        if(paymentcompleted.isMe()){
            System.out.println("====================================결제완료 1차====================================");
            if(reservationManagementrepository.findById(paymentcompleted.getReservationNumber()) != null){
                System.out.println("====================================결제완료====================================");
                ReservationManagement reservationManagement = reservationManagementrepository.findById(paymentcompleted.getReservationNumber()).get();
                reservationManagement.setReserveStatus("paymentComp");
                reservationManagementrepository.save(reservationManagement);
            }

        }

    }

}

```

객실 시스템은 예약/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 객실시스템이 유지보수로 인해 잠시 내려간 상태라도 예약을 받는데 문제가 없다:
```
# 객실 서비스 (room) 를 잠시 내려놓음 (ctrl+c)

#예약처리
http post localhost:8081/reservationManagements  customerName="Lee" customerId=123 reserveStatus="1" roomNumber=1 paymentPrice=50000
   #Success
http post localhost:8081/reservationManagements  customerName="Lee" customerId=123 reserveStatus="1" roomNumber=1 paymentPrice=50000
   #Success

#객실상태 확인
http localhost:8084/roomInfos     # 객실 안바뀜 확인

#객실 서비스 기동
cd RoomManagement
mvn spring-boot:run

#객실상태 확인
http localhost:8084/roomInfos     # 모든 주문의 상태가 "배송됨"으로 확인
```

## CQRS 패턴 
사용자 View를 위한 객실 정보 조회 서비스를 위한 별도의 객실 정보 저장소를 구현
- 이를 하여 RoomInfo 서비스를 별도로 구축하고 저장 이력을 기록한다.
- 모든 정보는 비동기 방식으로 호출한다.

```
RoomManagement.java(Entity)

@Entity
@Table(name="RoomManagement_table")
public class RoomManagement {
    ...

    @PostPersist
    public void onPostPersist(){
        RoomConditionChanged roomConditionChanged = new RoomConditionChanged();
        roomConditionChanged.setRoomNumber(this.getRoomNumber());
        roomConditionChanged.setRoomStatus(this.getRoomStatus());
        BeanUtils.copyProperties(this, roomConditionChanged);
        roomConditionChanged.publishAfterCommit();
    }
    @PostUpdate
    public void onPostUpdate(){
            RoomConditionChanged roomConditionChanged = new RoomConditionChanged();
            roomConditionChanged.setRoomNumber(this.getRoomNumber());
            roomConditionChanged.setRoomStatus(this.getRoomStatus());
            BeanUtils.copyProperties(this, roomConditionChanged);
            roomConditionChanged.publishAfterCommit();
    }
```
- RoomInfo에 저장하는 서비스 정책 (PolicyHandler)구현
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
                roomInfo.setRoomStatus(roomConditionChanged.getRoomStatus());
                roomInfoRepository.save(roomInfo);
        }
    }
}

```

# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AZURE를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 azure-pipelines.yml 에 포함되었다.

![image](https://user-images.githubusercontent.com/36434874/81782632-17017c00-9535-11ea-9562-9750c1bfd24e.png)

![image](https://user-images.githubusercontent.com/36434874/81782849-65af1600-9535-11ea-8c64-41e6a80ada77.png)

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 예약(Reservation)-->결제(Payment) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제:PaymentManagement) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (PaymentManagement) Payment.java (Entity)

    @PrePersist
    public void onPrePersist(){  //결제이력을 저장한 후 적당한 시간 끌기

        ...
        
        try {
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
                System.out.println("=============결재 승인 완료=============");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
    }
```


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy paymentmanagement --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://paymentmanagement:8080/payments POST {"reservationStatus": "reserve"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 


__As-IS__
```
Transactions:                    642 hits
Availability:                  51.16 %
Elapsed time:                  48.78 secs
Data transferred:               0.68 MB
Response time:                  4.39 secs
Transaction rate:              13.16 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   57.80
Successful transactions:         642
Failed transactions:             613
Longest transaction:            8.91
Shortest transaction:           0.00
```
__TO-BE__
```
Transactions:                   2293 hits
Availability:                  99.78 %
Elapsed time:                 119.58 secs
Data transferred:               0.70 MB
Response time:                  4.58 secs
Transaction rate:              19.18 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   87.80
Successful transactions:        2293
Failed transactions:               5
Longest transaction:           15.44
Shortest transaction:           0.45
```

### 개별과제 주제

신규 서비스를 MSA로 추가할 수있는가

### 개별과제 분석 설계

-- To-BE 조직
![image](https://user-images.githubusercontent.com/25599335/81894577-0f041380-95eb-11ea-957d-3098c0076ffc.png)



### 개별과제용 요구사항 도출

기능적 요구사항
1. 고객이 결재시 결재금액의 1% 마일리지를 적립한다
2. 적립한 마일리지의 금액을 사용하여 결재를 할수 있다.

비기능적 요구사항
1. 결재시 마일리지 적립은 동기식으로 구현한다.
2. 마일리지를 사용하여 결재도 동기식으로 구현한다.

    
### 개별과제용 신규 모델링 추가

![81891789-55a23f80-95e4-11ea-9926-e6d4260bf944](https://user-images.githubusercontent.com/25599335/81894309-681f7780-95ea-11ea-8aaa-a5c109ca77e8.png)


### 개별과제  기능적/비기능적 요구사항을 커버하는지 검증

기능적 요구사항
1. 고객이 결재시 결재금액의 1% 마일리지를 적립한다
2. 적립한 마일리지의 금액을 사용하여 결재를 할수 있다.

비기능적 요구사항
1. 결재시 마일리지 적립은 비동기식으로 구현한다.
2. 마일리지를 사용하여 결재 동기식으로 구현한다.

### 헥사고날 아키텍처 다이어그램 도출

![image](https://user-images.githubusercontent.com/25599335/81893018-8768d580-95e7-11ea-8b20-61299296d026.png)

    - Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

### 구현:
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다

### DDD 의 적용
각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다

```
package hotelmanage;


import javax.persistence.*;
import org.springframework.beans.BeanUtils;

@Entity
@Table(name="Mileage_table")
public class Mileage {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private int Mileageid; // 생성
    private int PaymentId; // 생성
    private int PaymentPrice; // 받아야되고
    private String PaymentStatus;//결재될떄 "Y" 로 셋팅
    private int Mileage;

    @PrePersist
    public void onPrePersist() {

            System.out.println("=============마일리지 적립 처리중=============");
            MileageSaved mileageSaved = new MileageSaved();

            mileageSaved.setPaymentId(PaymentId);
            System.out.printf("PaymentId : %d\n", PaymentId);
            mileageSaved.setPaymentPrice(PaymentPrice);
            System.out.printf("PaymentPrice : %d\n", PaymentPrice);
            mileageSaved.setPaymentStatus(PaymentStatus);
            System.out.printf("PaymentStatus : %s\n", PaymentStatus);

            if("Y".equals(PaymentStatus)) {
                Mileage = PaymentPrice / 100;
            }else{
                Mileage = 0;
            }

            System.out.printf("Mileage : %d\n", Mileage);

            BeanUtils.copyProperties(this, mileageSaved);
            mileageSaved.publishAfterCommit();

            try {
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
                System.out.println("=============마일리지 적립 완료=============");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

    }

    public int getMileageid() {
        return Mileageid;
    }

    public void setMileageid(int mileageid) {
        Mileageid = mileageid;
    }

    public int getPaymentId() {
        return PaymentId;
    }

    public void setPaymentId(int paymentId) {
        PaymentId = paymentId;
    }

    public int getPaymentPrice() {
        return PaymentPrice;
    }

    public void setPaymentPrice(int paymentPrice) {
        PaymentPrice = paymentPrice;
    }

    public String getPaymentStatus() {
        return PaymentStatus;
    }

    public void setPaymentStatus(String paymentStatus) {
        PaymentStatus = paymentStatus;
    }

    public int getMileage() {
        return Mileage;
    }

    public void setMileage(int mileage) {
        Mileage = mileage;
    }
}
```

 - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring     
   Data REST 의 RestRepository 를 적용하였다

```
package hotelmanage;

import org.springframework.data.repository.PagingAndSortingRepository;


public interface MileageRepository extends PagingAndSortingRepository<Mileage, Long> {
}
````

  - 로컬 테스트 진행 개별 모듈 
  
  ![image](https://user-images.githubusercontent.com/25599335/81893554-e24efc80-95e8-11ea-97db-16ead5d62bcd.png)
  
  ![image](https://user-images.githubusercontent.com/25599335/81893683-0579ac00-95e9-11ea-9a3b-acf9907c1f55.png)

