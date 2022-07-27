# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
   mvn spring-boot:run
```

## SAGA Pattern
쓰기에 있어 Pub-Sub의 Saga 패턴이 사용되었다.

방 등록 -> 예약 요청(고객) -> 예약 승인(집주인) -> 결제 진행 -> 예약 확인 -> 방 예약 완료

![image](https://user-images.githubusercontent.com/37835544/181160084-e0ea78b2-3e61-4ae1-8515-7c2923fd8f3c.png)


## CQRS Pattern

집주인 및 고객이 현황을 조회 할 수 있도록 viewPage를 CQRS 로 구현하였다.
- room, reservation, payment 개별 Aggregate 데이터 조회가 가능하다.
- 비동기식으로 처리되어 발행된 이벤트 기반 Kafka 를 통해 수신/처리 되어 별도 Table 에 관리한다
- Table 모델링 (ROOMVIEW)

- viewpage MSA ViewHandler 를 통해 구현 ("RoomRegistered" 이벤트 발생 시, Pub/Sub 기반으로 별도 Roomview 테이블에 저장)

![image](https://user-images.githubusercontent.com/37835544/181169708-77fa4897-7f56-4731-94a9-c3f5a0be3905.png)

- Event별로 Table 데이터 변경

![image](https://user-images.githubusercontent.com/37835544/181169941-100c3f8d-b7ec-40a4-808a-d1b6b33fb5d8.png)



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


- 결제 (payment) 서비스 중단 시 결과

![image](https://user-images.githubusercontent.com/37835544/181162844-f2ec96d1-2ab0-48f3-97f5-9a6c962e240b.png)

- 결제 서비스 실행 후 결과

![image](https://user-images.githubusercontent.com/37835544/181163106-417a5721-55c6-4347-b454-8583fe66bb5e.png)
