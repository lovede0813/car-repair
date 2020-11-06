
# automechanicsmall
# Table of contents

- [automechanicsmall](#---)
  - [서비스 시나리오](#서비스-시나리오)
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

# 서비스 시나리오

기능적 요구사항
1. 접수처에서 수리 신청을 받는다.
1. 수리처에서 접수처에 접수된 수리를 받고 접수상태를 변경한다.
1. 수리처에서 수리 완료한 후 접수상태를 변경한다.
1. 접수처에서 수리 완료된 접수에 대해 결제 요청을 한다.
1. 고객이 결제하고 접수상태를 변경한다.
1. 접수처에서 수리 취소를 요청한다.
1. 수리처에서 취소 요청된 수리를 취소하고 접수상태를 변경한다.
1. 수리처에서 접수처에 접수된 수리를 거절하고 접수상태를 변경한다.

비기능적 요구사항
1. 트랜잭션
    1. 수리가 완료된 수리접수건에 대해서만 결제가 성립되어야 한다 Sync 호출
1. 장애격리
    1. 수리처 기능이 수행되지 않더라도 접수은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
    1. 결제시스템에 문제가 발생했을 때 결제를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
1. 성능
    1. 접수처에서 수시로 접수상태를 확인할 수 있어야 한다. CQRS
    1. 수리처에서 상태를 변경할 수 있어야 한다.
    1. 고객이 직접 결제할 수 있어야 한다.

# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/n3taAaAWRZfsi0zRvKkRC4JRKAz1/mine/9b369712b03dfd02499b960c41add2c5/-MLLqWOYSX159CuLRqpu

### 이벤트 도출
![image](https://user-images.githubusercontent.com/22365716/98182203-8a1a0700-1f48-11eb-964e-636005c3aa9c.png)

### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/22365716/98183397-64423180-1f4b-11eb-86c6-2fd6434f676f.png)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/22365716/98183508-9a7fb100-1f4b-11eb-9843-a54d01660bec.png)

### 바운디드 컨텍스트로 묶기
![image](https://user-images.githubusercontent.com/22365716/98182436-19271f00-1f49-11eb-9bd9-7ed7a1958a3e.png)

    - 도메인 서열 분리 
        - Core Domain:  receipt : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 receipt 의 경우 1개월 1회 미만
        - Supporting Domain:  repair : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)
![image](https://user-images.githubusercontent.com/22365716/98183782-40cbb680-1f4c-11eb-8cfd-91c7ea098d87.png)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![image](https://user-images.githubusercontent.com/22365716/98183829-5b9e2b00-1f4c-11eb-8784-429e1dc55cf5.png)

### 완성된 1차 모형
![image](https://user-images.githubusercontent.com/22365716/98187334-06661780-1f54-11eb-82db-4351f052c5c9.png)


### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/46660236/98188908-3cf16180-1f57-11eb-80ef-86588ece7db9.png)

    - 접수처에서 수리 신청을 받는다. (ok)
    - 수리처에서 접수처에 접수된 수리를 받고 접수상태를 변경한다. (ok)
    - 수리처에서 수리 완료한 후 접수상태를 변경한다. (ok)
    - 접수처에서 수리 완료된 접수에 대해 결제 요청을 한다. (ok)
    - 고객이 결제하고 접수상태를 변경한다. (ok)

![image](https://user-images.githubusercontent.com/46660236/98188999-6c07d300-1f57-11eb-91e3-73521fa58498.png)

    - 접수처에서 수리 취소를 요청한다. (ok)
    - 수리처에서 취소 요청된 수리를 취소하고 접수상태를 변경한다. (ok)

![image](https://user-images.githubusercontent.com/46660236/98188993-6a3e0f80-1f57-11eb-90aa-053abdad6703.png)

    - 수리처에서 접수처에 접수된 수리를 거절하고 접수상태를 변경한다. (ok)


### 모델 수정

![모델수정](https://user-images.githubusercontent.com/46660236/98321246-f0298b80-2027-11eb-919f-b1ad830dbe6c.PNG)
    
    - 수정된 모델은 모든 요구사항을 커버함.

### 비기능 요구사항에 대한 검증

![모델수정](https://user-images.githubusercontent.com/46660236/98321246-f0298b80-2027-11eb-919f-b1ad830dbe6c.PNG)

    - 트랜잭션 처리
        - 결제 완료시 배송에 대해서는 Request-Response 방식 처리

## 헥사고날 아키텍처 다이어그램 도출
    
![헥사고날](https://user-images.githubusercontent.com/46660236/98321195-c8d2be80-2027-11eb-8c71-20abffd8c466.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 리액트으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)
```
cd receipt
mvn spring-boot:run
cd repair
mvn spring-boot:run
cd payment
mvn spring-boot:run
cd display
mvn spring-boot:run
cd delivery
mvn spring-boot:run
```
## DDD 의 적용
- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.
```
package automechanicsmallkde;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Delivery")
public class Delivery {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String stat;
    private Long receiptId;
    private String vehiNo;

    @PrePersist
    public void onPrePersist(){
        try {
            Thread.currentThread().sleep((long) (800 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @PostPersist
    public void onPostPersist(){
        System.out.println("###delivery.java - PrePersist###");
        System.out.println(this.getReceiptId());
        DeliveryStarted deliveryStarted = new DeliveryStarted();
        BeanUtils.copyProperties(this, deliveryStarted);
        //paymentApproved.setPayAmt(this.getPayAmt());
        //paymentApproved.setReceiptId(this.getReceiptId());

        deliveryStarted.publishAfterCommit();
    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getStat() {
        return stat;
    }

    public void setStat(String stat) {
        this.stat = stat;
    }

    public Long getReceiptId() {
        return receiptId;
    }

    public void setReceiptId(Long receiptId) {
        this.receiptId = receiptId;
    }

    public String getVehiNo() {
        return vehiNo;
    }

    public void setVehiNo(String vehiNo) {
        this.vehiNo = vehiNo;
    }

}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package automechanicsmallkde;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface DeliveryRepository extends PagingAndSortingRepository<Delivery, Long>{

}
```
- 적용 후 REST API 의 테스트
```
# 접수 생성 & 수리 요청
curl http://localhost:8081/receipts -H "Content-type:application/json" -X POST -d "{\"vehiNo\":\"0101\", \"stat\":\"REPAIRREQUEST\"}"
# 배송 요청
curl http://localhost:8081/receipts/1 -H "Content-type:application/json" -X PATCH -d "{\"stat\":\"DELIVERYREQUESTED\"}"
# 각 어그리게이트 확인 (접수, 배송 시작)
http http://localhost:8088/receipts/1
http http://localhost:8088/deliveries/1
```
## 폴리글랏 퍼시스턴스
마이크로서비스의 폴리그랏 퍼시스턴스의 예로 데이터의 빈번한 입출력을 사용하는 부분은 Display(View)의 저장소는 Mongo DB(NO SQL)를 사용하고 그 외의 업무 도메인인 접수(Receipt), 수리(Repair), 결재(Payment), 배송(Delivery)는 Maria DB(RDB)를 사용하였다.
application.yml의 간단한 설정을 통해 설정이 가능하다.
```
application.xml (delivery service)
spring:
  profiles: default
  jpa:
    database-platform: org.hibernate.dialect.MariaDB53Dialect
    hibernate:
      ddl-auto: create #create update none
    properties:
      hibernate:
        show_sql: true
        format_sql: true
  datasource:
    url: jdbc:mariadb://${delivery.db.url}:3306/delivery?useSSL=true
    username: ${delivery.db.name}
    password: ${delivery.db.password}
    driverClassName: org.mariadb.jdbc.Driver
application.xml (display service)
spring:
  profiles: default
  data:
    mongodb:
      uri: mongodb://${display.db.url}:27017/display
      username: ${display.db.name}
      password: ${display.db.password}
  jpa:
    properties:
      hibernate:
        show_sql: true
        format_sql: true
```
## 동기식 호출 과 Fallback 처리
분석단계에서의 조건 중 하나로 결제(payment)->배송(delivery) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.
- 배송서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현
```
# (delivery) DeliveryService.java
package automechanicsmall.external;

import automechanicsmall.external.Delivery;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="delivery", url="${api.delivery.url}")
public interface DeliveryService {

    @RequestMapping(method= RequestMethod.POST, path="/deliveries")
    public void deliveryStart(@RequestBody Delivery delivery);

}
- 배송 시작 후(@PostPersist) 정상적인 서비스일 경우 pub/sub 방식으로 receipt Stat 변경

# Delivery.java (Entity)
    @PostPersist
    public void onPostPersist(){
        System.out.println("###delivery.java - PrePersist###");
        System.out.println(this.getReceiptId());
        DeliveryStarted deliveryStarted = new DeliveryStarted();
        BeanUtils.copyProperties(this, deliveryStarted);
        //paymentApproved.setPayAmt(this.getPayAmt());
        //paymentApproved.setReceiptId(this.getReceiptId());

        deliveryStarted.publishAfterCommit();
    }
```
- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 배송 시스템이 장애가 나면 배송요청을 못받는다는 것을 확인:
```
# 배송 (delivery) 서비스를 잠시 내려놓음 (ctrl+c)
#배송 요청
curl http://localhost:8081/receipts/4 -H "Content-type:application/json" -X PATCH -d "{\"stat\":\"DELIVERYREQUESTED\"}"   #Fail
#delivery 재기동
cd delivery
mvn spring-boot:run
#주문처리
curl http://localhost:8081/receipts/4 -H "Content-type:application/json" -X PATCH -d "{\"stat\":\"DELIVERYREQUESTED\"}"   #Success
```
## 비동기식 호출 / 장애격리 / 최종 (Eventual) 일관성 테스트
수리 프로세스에 문제가 있더라도 접수는 계속 받을 수 있도록 비동기식 호출하여 처리 한다. (kafka)
추후에 수리 프로세스가 복구 완료되면 접수에서 정상적으로 수리 접수 되었다고 이벤트를 수신한다. (Polish Hanler 처리 및 kafka로 접수에 배송 요청 상태 변경 회신)
```
<receipt.java>
@PostPersist
public void onPostPersist() throws Exception {
    System.out.println("###Reservaton.java - onPrePersist###");
    try {
        // 수리 요청
        if(this.stat.equals("REQUESTREPAIR")) {
            Received received = new Received();
            BeanUtils.copyProperties(this, received);
            received.setReceiptId(this.getId());
            received.publishAfterCommit();
        }else{
            //Exception exception = new Exception();
            //throw exception; //예외 발생
        }
    }catch(Exception e) {
        throw new Exception("요청할 수 없는 상태 입니다.");
        //return;
    }finally {
    }
}
<Repair - PolicyHandler>
@Service
public class PolicyHandler{
    @Autowired
    RepairRepository repairRepository;
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReceived_RequestRepair(@Payload Received received){
        if(received.isMe()){
            System.out.println("##### listener RequestRepair : " + received.toJson());
            Repair repair = new Repair();
            repair.setReceiptId(received.getReceiptId());
            repair.setVehiNo(received.getVehiNo());
            repair.setStat("REQUESTREPAIR");
            repairRepository.save(repair);
        }
    }
}
<Repair.java>
@PostPersist
public void onPostPersist(){
    System.out.println("###Repair.java - onPostPersist###");
    // 수리 요청
    if(this.getStat().equals("REQUESTREPAIR")) {
        RepairReceived repairReceived = new RepairReceived();
        BeanUtils.copyProperties(this, repairReceived);
        System.out.println("####"+this.getReceiptId()+"####");
        repairReceived.publishAfterCommit();
    }
}
```
 시스템은 접수/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 수리시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# 수리 서비스 (store) 를 잠시 내려놓음 (ctrl+c)
#처리
http POST http://localhost:8088/receipts vehiNo=1234 stat=REQUESTREPAIR   #Success
http POST http://localhost:8088/receipts vehiNo=4567 stat=REQUESTREPAIR   #Success
#주문상태 확인
http GET http://localhost:8088/receipts     # 접수 완료로 변경되지 않음
http GET http://localhost:8088/repairs      # 수리 정보 없음
#수리 서비스 기동
cd repair
mvn spring-boot:run
#수리상태 확인
http GET http://localhost:8088/receipts     # 접수 완료로 변경
http GET http://localhost:8088/repairs      # 수리 정보 생성
```

# 운영


## 동기식 호출 / 서킷 브레이킹 / 장애격리
* 서킷 브레이킹 프레임워크의 선택: Istio Destination rule
MSA 의 각 서비스들의 에러가 전파되는 것을 막기 위해서 특정 서비스에서 에러가 발생할 경우 해당 서비스로의 연결을 차단하도록 구성하였습니다.
- Destination rule 를 설정:  5xx Error 가 5번 연속적으로 발생 시 해당 서비스로의 연결을 15분 동안 끊는다.
```
# kubectl -n automechanic get destinationrule
NAME         HOST      AGE
display-dr   display   18h
payment-dr   payment   19h
receipt-dr   receipt   19h
repair-dr    repair    19h
```
```
# payment-dr.yml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-dr
  namespace: automechanic
spec:
  host: payment
  trafficPolicy:
    outlierDetection:
      baseEjectionTime: 15m
      consecutive5xxErrors: 5
      interval: 5m
      maxEjectionPercent: 100
```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 63.55% 가 성공하였고, 46%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 90프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy delivery --min=1 --max=10 --cpu-percent=90
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 'http://40.82.137.107:8080/deliveries GET
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy delivery -w
```
- 어느정도 시간이 흐른 후 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
delivery   1/1     1            1           45h
delivery   1/2     1            1           45h
delivery   1/2     1            1           45h
delivery   1/2     1            1           45h
delivery   1/2     2            1           45h
:
```
