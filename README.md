

#### 커피주문 마이크로서비스 msa-service-coffee-order

#### 회원확인 마이크로서비스 msa-service-coffee-member

#### 주문처리상태확인 마이크로서비스 msa-service-coffee-status

======================================================

커피주문시 회원정보확인, 주문데이터저장, 주문처리상태확인 마이크로서비스에 주문내역을 큐잉시스템으로 전송

커피주문 마이크로서비스는 큐잉시스템을 이용해 메시지발행, 주문처리상태확인 마이크로스비스는 큐잉서비스를 이용해 메시지 구독

## 커피주문 마이크로서비스 msa-service-coffee-order

커피주문후 주문내역을 실시간 알려주는 이벤트메시지처리 관련 kafka (마이크로서비스간 데이터연계) 를 build.gradle에 추가

카프카설치하면 주키퍼가 설치됨. 주키퍼 기동후 카프가기동. 

큐잉시스템은 마이크로서비스간 메시지발행, 구독 메커니즘 제공하여 마이크로서비스간 직접적 호출을 대신하는 느슨한관계유지

1.domain(업무구현)

-model : 

OrderEntity (엔티티) 

CoffeeOrderCVO (밸류오브젝트)- 클라이언트가 백엔드서비스호출시 {"orderNumber":"1"} JSON형식으로 보내면 CVO에서 orderNumber변수로 대응

-repository

ICoffeeOrderRepository -커피주문데이터의 CRUD관련 인터페이스 정의 (public String coffeOrderSave(OrderEntity orderEntity) -커피주문정보저장

-service

ICoffeeOrder -커피주문처리비즈니스로직 

(public String coffeeOrder(CoffeeOrderCVO coffeOrderCVO);

OrderEntity.setOrderNumber(coffeeOrderCVO.getOrderNumber());  //CVO에서 전달받은 변수를 Entity에 set해주고

iCoffeeOrderRepository.coffeOrderSave(orderEntity);		//repository에 엔티티를 저장해줌

)

2.springboot (기술구현)

-configuration - 카프카서버 9092포트 설정

-msesageq -커피주문정보를 주문처리상태확인 마이크로서비스에 전달하기위해 큐잉시스템 사용설정

KafkaProducer - JPA기반저장 

KafkaProducerConfig

-repository

OrderEntityJPO (extends OrderEntity)

CoffeeOrderRepository

(iCoffeeOrderRepository를 상속받아 coffeeOrderSave구현 - orderEntityJPO.setOrderNumber(odrerEntity.getOrderNumber());

iCoffeeOrderJpaRepository.save(orderEntityJPO);

)

-rest -커피주문요청처리 (회원정보확인, 주문데이터저장, 주문처리상태확인 마이크로서비스에 주문내역을 큐잉시스템으로 전송)

coffeeOrderRestController (

iMsaServiceCoffeeMember.coffeeMember(coffeOrderCVO.getCustomerName()); -회원정보확인

CoffeeOrderServiceImpl.coffeeOrder(coffeeOrderCVO); -주문데이터저장

kafkaProducer.send("honny-kafka-test", coffeeOrderCVO); -큐잉시스템으로 주문처리상태확인 마이크로서비스에 주문내역전송

);

-service

CoffeeOrderServiceImpl

IMSAServiceCoffeeMember( - 회원확인마이크로서비스 호출하여 회원정보얻기위해 FeignClient를 이용

@FeignClient("msa-service-coffee-member")

public interface IMsaServiceCoffeeMember

);

===============================

## 회원확인 마이크로서비스 msa-service-coffee-member

messageq 패키지 - kafka로부터 전달된 메시지 수신하는 메시지 소비자 구현

springboot

-configuration

WebConfiguration

-repository (회원정보조회기능 -마이바티스로처리)

MemberDVO ( 

ICoffeeMemberMapper (MemberDVO existsByMemberName(MemberDVO memberDVO);)

-rest

CoffeeMemberRestController (

@HystrixCommand //회원확인

@RequestMapping (value = "/coffeeMember/ 

if(iCoffeeMemberMapper.existsByMemberName(memberDVO));

@HystrixCommand(fallbackMethod = "fallbackFunction") //서킷브레이커 테스트

@RequestMapping(value = "/fallbackTest", )

public String fallbackTest() throws Throwable {

  throw new Throwable("fallbackTest");
  
}

=================================
## 주문처리상태확인 마이크로서비스 msa-service-coffee-status

-configuration

WebConfiguration = 데이터베이스정보 설정

-repository -주문처리상태 조회

OrderStatusDVO

iCoffeeStatusMapper (OrderStatusDVO selectCoffeeOrderStatus();)

-messageq

KafkaConsumer클래스는 카프카큐로부터 메시지감지후 수신처리 (@KafkaListener로 메시지감지)

KafkaConsumerConfig 카프카큐에 접속하기위한 설정, 메시지구독 리스너 설정

-rest

CoffeeOrderStatusRestController (

@HystrixCommand  //주문처리상태확인

@RequestMapping(value="/coffeeOrderStatus", method=RequestMethod.POST)

public ResponseEntity<OrderStatusDVO> coffeeOrderStatus(){

  OrderStatusDVO orderStatusDVO = iCoffeeStatusMapper.selectCofeeOrderStatus();
  
  return new ResponseEntity<OrderStatusDVO>(orderStatusDVO, HttpStatus.OK);
  
)

=============================================
### 유레카서버

MsaEurekaServerApplication에 @EnableEurekaServer 등록

마이크로서비스감지하기위해 MicroServerApplication에 @EnableEurekaClient 등록

application.yml에 등록

스프링유레카 웹화면에서 확인가능

### 줄서버 (부하분산, 라우팅)

application.yml에 zuul: 등록

application.yml에 각 마이크로서비스의 고유이름 정의. 서비스기동시 유레카서버에등록, 줄서버가 이름이용해서 서비스라우팅

### 히스트릭스서버

히스트릭스 클라이언트 설치된 마이크로서비스는 호출시 스트림메시지를 터빈서버에 전송

터빈서버이용하여 개별 히스트릭스 스트림을 한번에 수집

히스트릭스대시보드는 터빈서버에 연결하여 일괄취합된 스트림을 웹화면으로 확인

==================================================

### 프로젝트 실행법

//카프카 kafka_2.13-3.6.0 설치후 테스트

//설치경로로이동후 실행

cd kafka_2.13-3.6.0

cd bin

cd windws

zookeeper-server-start.bat ../../config/zookeeper.properties

kafka-server-start.bat ../../config/server.properties

//토픽생성

kafka-topics.bat --create --topic quickstart-events --bootstrap-server localhost:9092

(결과)Created topic quickstart-events.

//토픽생성잘됬는지 확인

kafka-topics.bat --list --bootstrap-server localhost:9092

//프로듀서로 토픽에 이벤트쓰기

kafka-console-producer.bat --topic quickstart-events --bootstrap-server localhost:9092

//컨슈머로 토픽의 이벤트읽기

kafka-console-consumer.bat --topic quickstart-events --from-beginning --bootstrap-server localhost:9092

(입력한값 불러오는것 확인됨)

https://kjchoi.co.kr/24
=======================================
//프로젝트오류 - build path, jdk, jre 버전 설정 차이떄문 / 이클립스 프로젝트에서 import - gradle build해줘야함

//서버실행포트 - 각 프로젝트 7개를 스프링부트로 옮겨서 실행함

9091 유레카
9090 줄
9999 터빈
9093 히스트릭스
8080 커피주문
8081 회원확인
8082 주문처리상태확인 

//주키퍼 서버가동 (위에있음)

//카프카 서버가동 (위에있음)

//테이블, 데이터생성 

(put) http://localhost:9090/coffeeMember/createMemberTable

(put) http://localhost:9090/coffeeMember/insertMemberData

//주문처리상태확인

(put) http://localhost:9090/coffeeStatus/createStatusTable

//포스트맨으로실험(커피주문)

http://localhost:9090/coffeeOrder/coffeeOrder

{"id":"","orderNumber":"1","coffeeName":"espresso","coffeeCount":"2","customerName":"kevin"}

//포스트맨으로실험(커피주문상태확인)

http://localhost:9090/coffeeStatus/coffeeOrderStatus

{
    "id": "1",
    "orderHistory": "{\"id\":\"\",\"orderNumber\":\"1\",\"coffeeName\":\"espresso\",\"coffeeCount\":\"2\",\"customerName\":\"kevin\"}"
}

//유레카 웹에 켜지는것 확인
http://localhost:9091/

//터빈서버
http://localhost:9999/turbine.stream


















