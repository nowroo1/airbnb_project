![image](https://user-images.githubusercontent.com/15603058/119284989-fefe2580-bc7b-11eb-99ca-7a9e4183c16f.jpg)

# 숙소예약(AirBnB)

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 숙소예약](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
   mvn spring-boot:run
```

## SAGA Pattern
쓰기에 있어 Pub-Sub의 Saga 패턴이 사용되었다.

방 등록 -> 예약 요청(고객) -> 예약 승인(주인) -> 결제 진행 -> 예약 확인 -> 방 예약 완료

![image](https://user-images.githubusercontent.com/37835544/181160084-e0ea78b2-3e61-4ae1-8515-7c2923fd8f3c.png)


## CQRS Pattern

주인 및 고객이 현황을 조회 할 수 있도록 viewPage를 CQRS 로 구현하였다.
- room, reservation, payment 개별 Aggregate 데이터 조회가 가능하다.
- 비동기식으로 처리되어 발행된 이벤트 기반 Kafka 를 통해 수신/처리 되어 별도 Table 에 관리한다
- Table 모델링 (ROOMVIEW)

  ![image](https://user-images.githubusercontent.com/77129832/119319352-4b198c00-bcb5-11eb-93bc-ff0657feeb9f.png)
- viewpage MSA ViewHandler 를 통해 구현 ("RoomRegistered" 이벤트 발생 시, Pub/Sub 기반으로 별도 Roomview 테이블에 저장)
  ![image](https://user-images.githubusercontent.com/77129832/119321162-4d7ce580-bcb7-11eb-9030-29ee6272c40d.png)
  ![image](https://user-images.githubusercontent.com/31723044/119350185-fccab400-bcd9-11eb-8269-61868de41cc7.png)
- 실제로 view 페이지를 조회해 보면 모든 room에 대한 전반적인 예약 상태, 결제 상태, 리뷰 건수 등의 정보를 종합적으로 알 수 있다
  ![image](https://user-images.githubusercontent.com/31723044/119357063-1b34ad80-bce2-11eb-94fb-a587261ab56f.png)


## Gateway
      - gateway 스프링부트 App을 추가 후 application.yaml내에 각 마이크로 서비스의 routes 를 추가하고 각 서버의 URI 작성
       
          - application.yaml 예시
            ```
		spring:
		  profiles: default
		  cloud:
		    gateway:
		      routes:
			- id: room
			  uri: http://localhost:8081
			  predicates:
			    - Path=/rooms/**, /reviews/** 
			- id: payment
			  uri: http://localhost:8082
			  predicates:
			    - Path=/payments/** 
			- id: reservation
			  uri: http://localhost:8083
			  predicates:
			    - Path=/reservations/** 
			- id: Message
			  uri: http://localhost:8084
			  predicates:
			    - Path=/messages/** 
			- id: viewpage
			  uri: http://localhost:8085
			  predicates:
			    - Path= /roomviews/**
			- id: frontend
			  uri: http://localhost:8080
			  predicates:
			    - Path=/**
	    
            ```


# Correlation/Compensation(Unique Key)

PolicyHandler에서 처리 시 어떤 건에 대한 처리인지를 구별하기 위한 Correlation-key 구현을 
이벤트 클래스 안의 변수로 전달받아 서비스간 연관된 처리를 정확하게 구현하고 있습니다. 

예약(Reservation)을 하면 동시에 연관된 방(Room), 결제(Payment) 등의 서비스의 상태가 적당하게 변경이 되고,
예약건의 취소를 수행하면 다시 연관된 방(Room), 결제(Payment) 등의 서비스의 상태값 등의 데이터가 적당한 상태로 변경되는 것을
확인할 수 있습니다.


예약 전 - 방 상태(status = true)

![image](https://user-images.githubusercontent.com/37835544/181166258-2c4b0d4e-0d5a-44ec-87eb-4ef4551bf410.png)


예약 후 - 방 상태(status = false)

![image](https://user-images.githubusercontent.com/37835544/181166377-340c6fe8-0c62-49ef-abc9-92c0d19b576b.png)


예약 후 - 결제 상태(payments/1 존재)

![image](https://user-images.githubusercontent.com/37835544/181166457-c2a9ee2f-13de-4137-83b3-f193b1ac267f.png)


예약 취소 - 방 상태(status = true)

![image](https://user-images.githubusercontent.com/37835544/181167738-7e592ddd-62f2-42c7-bae3-488c23cae1e2.png)


예약 취소 - 결제 상태(payment 삭제)

![image](https://user-images.githubusercontent.com/37835544/181166867-88016341-0ed6-43f4-8ef2-ffb8bb3ba062.png)



## Request/Response(Feign Client / Sync.Async)

ReservationAccepted 시 approvePayment 호출은 Req/Res 방식을 이용하였고, FeignClient 를 이용하여 처리하였다.

```
# PaymentService.java

@FeignClient(name = "payment", url = "${api.url.payment}")
public interface PaymentService {
    @RequestMapping(method = RequestMethod.POST, path = "/payments")
    public void approvePayment(@RequestBody Payment payment);
    // keep

}


```

- 예약 승인을 받은 직후 (@PostUpdate) 가능상태 확인 및 결제를 동기(Sync)로 요청하도록 처리
```
# Reservation.java (Entity)

    @PostPersist
    public void onPostUpdate(){
    
            ReservationAccepted reservationAccepted = new ReservationAccepted(this);
            reservationAccepted.publishAfterCommit();

            msaneil.external.Payment payment = new msaneil.external.Payment();
            // mappings goes here
            payment.setRsvId(this.getRsvId());
            payment.setRoomId(this.getRoomId());
            payment.setStatus(true);
            payment.setPayId(this.getRsvId());

            ReservationApplication.applicationContext
                .getBean(msaneil.external.PaymentService.class)
                .approvePayment(payment);
    }
```


# 결제 (payment) 서비스 중단 시 결과

![image](https://user-images.githubusercontent.com/37835544/181162844-f2ec96d1-2ab0-48f3-97f5-9a6c962e240b.png)

# 결제 서비스 실행 후 결과

![image](https://user-images.githubusercontent.com/37835544/181163106-417a5721-55c6-4347-b454-8583fe66bb5e.png)
