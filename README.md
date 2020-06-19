# 콘서트 예매 시스템

# Table of contents

- [서비스 시나리오](#서비스-시나리오)
  - [시나리오 테스트결과](#시나리오-테스트결과)
- [분석/설계](#분석설계)
- [구현](#구현)
  - [DDD 의 적용](#ddd-의-적용)
  - [Gateway 적용](#Gateway-적용)
  - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
  - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
  - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
- [운영](#운영)
  - [CI/CD 설정](#cicd설정)
  - [서킷 브레이킹 / 장애격리](#서킷-브레이킹-/-장애격리)
  - [오토스케일 아웃](#오토스케일-아웃)
  - [무정지 재배포](#무정지-재배포)
  

# 서비스 시나리오

기능적 요구사항
1. 관리자가 콘서트를 등록한다.
1. 사용자가 회원가입을 한다.
1. 사용자가 콘서트를 예약한다.
1. 사용자가 예약한 콘서트를 결제한다.
1. 결제가 완료되면 콘서트예약이 승인된다.
1. 콘서트예약이 승인되면 티켓 수가 변경된다. (감소)
1. 사용자가 예약 취소를 하면 결제가 취소된다.
1. 결제가 취소되면 티켓 수가 변경된다. (증가)
1. 사용자가 콘서트 예약내역 상태를 조회한다.

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 콘서트은 승인되지 않는다. > Sync 
1. 장애격리
    1. 콘서트 관리 기능이 수행되지 않아도 콘서트 예약은 수행된다. > Async (event-driven)
    1. Payment시스템이 과중되면 결제를 잠시후에 하도록 유도한다. > Circuit breaker, Fallback
1. 성능
    1. 사용자는 콘서트 예약내역을 확인할 수 있다. > CQRS


##시나리오 테스트결과

| 기능 | 이벤트 Payload |
|---|:---:|
| 관리자가 콘서트를 등록한다. | ![image](https://user-images.githubusercontent.com/62231786/85086806-aa099200-b216-11ea-8ca4-50eb47c3b02b.JPG) |
| 사용자가 회원가입을 한다. | ![image](https://user-images.githubusercontent.com/62231786/85086808-aa099200-b216-11ea-895e-3a7dcfeb4b71.JPG) |
| 사용자가 콘서트를 예약한다.</br>예약 시, 결제가 요청된다. | ![image](https://user-images.githubusercontent.com/62231786/85086809-aaa22880-b216-11ea-9d5c-fcf88fbd2a27.JPG) |
| 사용자가 예약한 콘서트를 결제한다.</br>결제가 완료되면 콘서트예약이 승인된다.</br>콘서트예약이 승인되면 티켓 수가 변경된다. (감소)| ![image](https://user-images.githubusercontent.com/62231786/85086811-aaa22880-b216-11ea-96aa-5ec29cd8a5d6.JPG) | 
| 사용자가 예약 취소를 하면 결제가 취소된다.</br>결제가 취소되면 티켓 수가 변경된다. (증가) | ![image](https://user-images.githubusercontent.com/62231786/85086805-a8d86500-b216-11ea-900a-be7c1555e61d.JPG) |
| 사용자가 콘서트 예약내역 상태를 조회한다. | [{"id":1,"bookingId":6659,"concertId":1,"userId":1,"status":"BookingRequested"},</br> {"id":2,"bookingId":6660,"concertId":3,"userId":1,"status":"PaymentCanceled"}] |

# 분석/설계

## Event Storming 결과

<img src="https://user-images.githubusercontent.com/62231786/85047444-ce408100-b1cc-11ea-805f-1c2557c986c5.png"/>

```
# 도메인 서열
- Core : Booking
- Supporting : Concert, User
- General : Payment
```


## 헥사고날 아키텍처 다이어그램 도출

* CQRS 를 위한 Mypage 서비스만 DB를 구분하여 적용
    
![image](https://user-images.githubusercontent.com/62231786/85047293-92a5b700-b1cc-11ea-9798-dfd58f79d993.png)


# 구현

## DDD 의 적용

분석/설계 단계에서 도출된 MSA는 총 5개로 아래와 같다.
* MyPage 는 CQRS 를 위한 서비스

| MSA | 기능 | port | 조회 API |
|---|:---:|:---:|:---:|
| Concert | 콘서트 관리 | 8081 | http://localhost:8081/cooncerts |
| Booking | 예약 관리 | 8082 | http://localhost:8082/bookings |
| Payment | 결제 관리 | 8083 | http://localhost:8083/payments |
| User | 사용자 관리 | 8084 | http://localhost:8084/users |
| MyPage | 콘서트 예약내역 관리 | 8086 | http://localhost:8085/bookingHistories |


## Gateway 적용

```
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: concert
          uri: http://concert:8080
          predicates:
            - Path=/concerts/** 
        - id: booking
          uri: http://booking:8080
          predicates:
            - Path=/bookings/** 
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/payments/** 
        - id: user
          uri: http://user:8080
          predicates:
            - Path=/users/** 
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path=/bookingHistories/**
```


## 폴리글랏 퍼시스턴스

CQRS 를 위한 Mypage 서비스만 DB를 구분하여 적용함. 인메모리 DB인 hsqldb 사용.

```
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->
		<dependency>
		    <groupId>org.hsqldb</groupId>
		    <artifactId>hsqldb</artifactId>
		    <version>2.4.0</version>
		    <scope>runtime</scope>
		</dependency>
```


## 동기식 호출 과 Fallback 처리

예약 > 결제 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리

- FeignClient 서비스 구현

```
# PaymentService.java

@FeignClient(name="payment", url="${feign.payment.url}", fallback = PaymentServiceFallback.class)
public interface PaymentService {
    @PostMapping(path="/payments")
    public void requestPayment(Payment payment);
}
```


- 동기식 호출

```
# Booking.java

@PostPersist
public void onPostPersist(){
    BookingRequested bookingRequested = new BookingRequested();
    BeanUtils.copyProperties(this, bookingRequested);
    bookingRequested.setStatus(BookingStatus.BookingRequested.name());
    bookingRequested.publishAfterCommit();

    Payment payment = new Payment();
    payment.setBookingId(this.id);

    Application.applicationContext.getBean(PaymentService.class).requestPayment(payment);
}
```


- Fallback 서비스 구현

```
# PaymentServiceFallback.java

@Component
public class PaymentServiceFallback implements PaymentService {

	@Override
	public void enroll(Payment payment) {
		System.out.println("Circuit breaker has been opened. Fallback returned instead.");
	}

}
```


## 비동기식 호출 과 Fallback 처리

- 비동기식 발신 구현

```
# Booking.java

@PostUpdate
public void onPostUpdate(){
    if (BookingStatus.BookingApproved.name().equals(this.getStatus())) {
        BookingApproved bookingApproved = new BookingApproved();
        BeanUtils.copyProperties(this, bookingApproved);
        bookingApproved.publishAfterCommit();
    }
}
```

- 비동기식 수신 구현

```
# PolicyHandler.java

@StreamListener(KafkaProcessor.INPUT)
public void paymentApproved(@Payload PaymentApproved paymentApproved){
    if(paymentApproved.isMe()){
	bookingRepository.findById(paymentApproved.getBookingId())
	    .ifPresent(
			booking -> {
				booking.setStatus(BookingStatus.BookingApproved.name());;
				bookingRepository.save(booking);
		    }
	    )
	;
    }
}
```


# 운영

## CI/CD 설정

- CodeBuild 기반으로 파이프라인 구성

<img src="https://user-images.githubusercontent.com/62231786/85087121-927ed900-b217-11ea-8f57-bbd4efc25997.JPG"/>

- Git Hook 연경

<img src="https://user-images.githubusercontent.com/62231786/85087123-93b00600-b217-11ea-90b3-4de01d03583a.JPG" />


## 서킷 브레이킹 / 장애격리

* Spring FeignClient + Hystrix 구현
* Booking 서비스 내 PaymentService FeignClient에 적용

- Hystrix 설정

```
# application.yml

feign:
  hystrix:
    enabled: true

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610
```

- 서비스 지연 설정

```
//circuit test
try {
    Thread.currentThread().sleep((long) (400 + Math.random() * 220));
} catch (InterruptedException e) { }
```

- 부하 테스트 수행

```
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8082/bookings/ POST {"concertId":1, "userId":1, "qty":5}'
```

- 부하 테스트 결과

```
2020-06-19 01:54:52.576[0;39m [32mDEBUG[0;39m [35m6600[0;39m [2m---[0;39m [2m[container-0-C-1][0;39m [36mo.s.c.s.m.DirectWithAttributesChannel   [0;39m [2m:[0;39m preSend on channel 'event-in', message: GenericMessage [payload=byte[142], headers={kafka_offset=4013, scst_nativeHeadersPresent=true, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@5775c5aa, deliveryAttempt=1, kafka_timestampType=CREATE_TIME, kafka_receivedMessageKey=null, kafka_receivedPartitionId=0, contentType=application/json, kafka_receivedTopic=sts, kafka_receivedTimestamp=1592499287785}]
Circuit breaker has been opened. Fallback returned instead.
Circuit breaker has been opened. Fallback returned instead.
[2m2020-06-19 01:54:52.576[0;39m [32mDEBUG[0;39m [35m6600[0;39m [2m---[0;39m [2m[o-8082-exec-153][0;39m [36mo.s.c.s.m.DirectWithAttributesChannel   [0;39m [2m:[0;39m postSend (sent=true) on channel 'event-out', message: GenericMessage [payload=byte[142], headers={contentType=application/json, id=cbdf4d07-547d-5dbe-80a1-659a0e00b607, timestamp=1592499291969}]
[2m2020-06-19 01:54:52.576[0;39m [32mDEBUG[0;39m [35m6600[0;39m [2m---[0;39m [2m[o-8082-exec-166][0;39m [36mo.s.c.s.m.DirectWithAttributesChannel   [0;39m [2m:[0;39m postSend (sent=true) on channel 'event-out', message: GenericMessage [payload=byte[142], headers={contentType=application/json, id=3a646994-497f-717a-cb13-443133007248, timestamp=1592499291969}]
```

```
defaulting to time-based testing: 30 seconds

{	"transactions":			         447,
	"availability":			      100.00,
	"elapsed_time":			       29.92,
	"data_transferred":		        0.10,
	"response_time":		        6.11,
	"transaction_rate":		       14.94,
	"throughput":			        0.00,
	"concurrency":			       91.21,
	"successful_transactions":	         447,
	"failed_transactions":		           0,
	"longest_transaction":		       17.07,
	"shortest_transaction":		        0.00
}
```


### 오토스케일 아웃

- 현재 상태 확인

```
  Namespace                   Name                        CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                        ------------  ----------  ---------------  -------------  ---
  default                     booking-7764c68d4b-27jrp    0 (0%)        0 (0%)      0 (0%)           0 (0%)         9h
  default                     concert-6b54bd565c-grvnf    0 (0%)        0 (0%)      0 (0%)           0 (0%)         15h
  default                     payment-684fd5785c-67ptn    0 (0%)        0 (0%)      0 (0%)           0 (0%)         9h
  kafka                       my-kafka-0                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         144m
  kafka                       my-kafka-zookeeper-2        0 (0%)        0 (0%)      0 (0%)           0 (0%)         16h
  kube-system                 aws-node-5rd64              10m (0%)      0 (0%)      0 (0%)           0 (0%)         16h
  kube-system                 coredns-555b56bfbb-mj2pk    100m (5%)     0 (0%)      70Mi (2%)        170Mi (6%)     16h
  kube-system                 kube-proxy-bmc2z            100m (5%)     0 (0%)      0 (0%)           0 (0%)         16h
```

- 오토스케일 설정
```
kubectl autoscale deploy booking --min=1 --max=3 --cpu-percent=1
```

- 부하 수행

```
siege -c100 -t60S -r10 --content-type "application/json" 'http://aa8dc72fe9cbb4ba0ba62c5720326102-1685876144.ap-northeast-2.elb.amazonaws.com:8080/bookings/ POST {"concertId":1, "userId":1, "qty":5}' -v
```

- 모니터링

```
kubectl get deploy booking -w
```

- 스케일 아웃 확인

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
booking   1/1     1            1           5h9m
```

```
defaulting to time-based testing: 60 seconds

{	"transactions":			        6316,
	"availability":			      100.00,
	"elapsed_time":			       60.00,
	"data_transferred":		        1.43,
	"response_time":		        0.94,
	"transaction_rate":		      105.27,
	"throughput":			        0.02,
	"concurrency":			       99.46,
	"successful_transactions":	        6316,
	"failed_transactions":		           0,
	"longest_transaction":		        6.22,
	"shortest_transaction":		        0.05
}
```


## 무정지 재배포

- API 호출을 통해 상태 확인

```
siege -c100 -t60S -r10 --content-type "application/json" 'http://a60f713056f2b477caf0532c3e37d2e3-1344816213.ap-northeast-2.elb.amazonaws.com:8080/bookingHistories' -v
```

- 호출 결과

```
Lifting the server siege...
Transactions:                  60364 hits
Availability:                  99.87 %
Elapsed time:                  99.02 secs
Data transferred:              26.53 MB
Response time:                  0.16 secs
Transaction rate:             609.61 trans/sec
Throughput:                     0.27 MB/sec
Concurrency:                   97.95
Successful transactions:       60364
Failed transactions:              80
Longest transaction:           29.22
Shortest transaction:           0.02
```

- POD 기동 확인

<img src="https://user-images.githubusercontent.com/62231786/85087355-497b5480-b218-11ea-804c-6e884f60c92f.JPG" />
