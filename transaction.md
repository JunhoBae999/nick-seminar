# Transaction?
여러가지 형태의 트랜잭션과 각각의 guarantee 들에 대해서 알아보고,

heterogeneous environments의 분산 트랜잭션을 다루기 위한 알고리즘과 프로토콜에 대해서 알아보고자 한다.

### 1.Transaction이란?
하나의 동작으로 실행되어야 하는 관련 동작들의 그룹 
작업의 논리적 단위 ← 외부에서는 그 결과가 전부 보이거나 혹은 아예 보이지 않음

→ 데이터 무결성 (integrity)를 보장
[image:68348D97-E9D0-4EB2-B14F-B993E6A751C4-8906-000313782DB8B4A6/CF303FE7-57A7-43DE-809D-2BB1338100E0.png]

ex) DB를 업데이트 하고 이벤트를 발행하는 것은 함께 수행되거나 혹은 아예 수행되지 않아야 함

### 2. Transaction의 역사
transaction은 관계형 데이터베이스의 발전과 밀접하게 연관되어 있음

> Edgar F.Codd, “The relational model of data"

#### 2.1  초기 Transaction 모델

##### ⓵ 관계형 데이터베이스는 편리함과 유연성을 기반으로 일상화됨.
→ 다중 사용자, 동시 접근 시스템에서의 복잡성을 초래함.
→ consistency에 대한 강제가 필요
→ ACID의 등장


##### ⓶ ACID를 준수하는 트랜잭션은 atomic & serializable 하도록 보장됨.
→ 따라서, 트랜잭션을 처리하는 시스템은 ACID를 준수하도록 설계됨
→ 짧은 실행 시간, 적은 수의 사용자, single database system에서는 효율적으로 작동
→ 하지만 수요가 급증함에 따라, 복잡도도 크게 상승
→ long-living and complex transactions의 필요성
→ complex transaction model의 등장 (sub-transaction, transaction groups) : long-living transaction이 실패하는 경우에 대해 더 정밀한 컨트롤 제공

#### 2.2 고급 Transaction 모델

##### ⓵ distributed & nested transaction 도입

* 어플리케이션이 복잡해 질 수록, 여러 데이터베이스 시스템에 대한 transactional access가 필요해짐
* distributed transaction : bottom-up / nested transactions : top-down
* 복잡한 트랜잭션을 subtransaction으로 분해
* 다수의 리소스에 대한 global integrity constraints 적용 가능 
→ 리소스 조차 상이해 짐에 따라 X/Open DTP (Distributed Transaction Processing) model 등장

##### ⓶ chained transaction, saga 도입
* nested transaction은 federated database system에서는 잘 작동했으나, long-live transaction에는 적합하지 않았음.
* chained transaction : 위와 같은 transaztion을 작고 순차적으로 수행되는 sub-transaction으로 분해
* saga : chained transaction을 기반으로, 이미 완료된 sub-transaction에 대해 roll back의 경우 보상 메커니즘을 적용, relaxed consistency (현재 MSA의 타당성 확인 가능하게 해줌)

#### 3. Local vs Distributed Transaction
[image:259BF3F8-16FC-4CC7-BA11-F542078E4BA6-8906-000313856F1ECECE/7D0E4B0E-A402-454E-A7B7-9BC502B8378B.png]

* 트랜잭션에 포함되는 operation은 한 곳의 (single-participating) 리소스에서 수행 될 수도,여러 곳의 리소스에서 (multiple-participating) 수행 될 수도 있음.
* transaction은 다수의 독립적인 resource (db, message queue, web service)와 관련있을 수도 있고,
* 각각의 resource는 동일한 VM에서 실행될 수도, 같은 PM의 다른 VM에서 실행될 수도, 서로 다른 PM에서 실행될 수도 있음
* resource의 수와 위치는 transaction을 구현할 때 어떤 것을 보장할 것이냐 (guarantees) 에 중요한 요소

### 4. Transaction Guarantees

Transaction을 쓰는 이유? data integrity를 위해

Data integrity? 모든 트랜잭션이 지원해야 하는 gurantee의 집합으로 정의할 수 있음. 

또한, 분산 시스템에서는 data paritioning을 더 효율적으로 하기 위해 위의 gurantee 중 어떤 것들을 포기해야 함.

#### 4.1 ACID Properties
[image:BFACEED4-83F5-4903-83B7-01FF05B473F7-8906-0003138FA8070D5B/k7ylxkrqn4cbh8kcrsi9.png]
* Atomicity: 데이터에 대한 변경을 모두 transaction의 일부가 되도록 하고, 그러한 변경점들을 하나의 엔티티/operation으로 만드는 것.즉, 변경을 모두 수행하거나 하나도 수행되지 않게 해야함.
* Consistency: 데이터에 대한 변경은 트랜잭션의 시작과 끝에서 consistent한 상태를 유지하면서 일어난다. 데이터의 consistent한 상태는 제약을 따라야 함
* Isolation: 트랜잭션의 중간 단계를 다른 트랜잭션이 알 수 없게 하는 것. 동시에 진행되는 트랜잭션들의 isolation level에 따라 다름. 
* Durability: 데이터가 영속화되면 다른 트랜잭션에 의해 revert 되지 못하는 것.

논쟁의 여지는 있지만, 트랜잭션이 꼭 위의 요소들을 보장해야하는 것은 아니다. 분산 시스템의 등장에 따라, 트랜잭션이란 용어가 위와는 다르게 조금더 자유롭게 사용되기도 한다. (availability에 중점을 둠)


#### Transaction In MySql

##### Locking? Transaction?
* Locking : ‘동시성’을 제어하는 기능
* Transaction : 데이터의 ‘정합성’을 보장하는 기능

##### MySQL에서의 트랜잭션 : InnoDB vs MyISAM

1. MyISAM 엔진, InnoDB 엔진 사용하는 각각의 테이블 생성, 1건 생성
``` mysql
create table tab_myisam(fdpk INT NOT NULL, PRIMARY KEY (fdpk)) ENGINE=MyISAM;

insert into tab_myisam(fdpk) values(3);

create table tab_innodb (fdpk INT NOT NULL, PRIMARY KEY (fdpk)) ENGINE=INNODB;

insert into tab_innodb(fdpk) values(3);
```

2. Auto-commit 모드에서 다음 insert 각각 실행
```mysql
SET autocommit = ON;

INSERT INTO tab_myisam(fdpk) values (1),(2),(3);
INSERT INTO tab_innodb(fdpk) values (1),(2),(3);
```

[image:459485C3-432B-4C28-B6FA-62D624903EB0-8906-000314E5FBA39EA7/D46678DB-689B-476C-8D84-BFC06C168054.png]

3. 조회 확인
[image:612C59F1-B442-43D8-93F2-63756820DA92-8906-000314EE3ACA7D4A/B4904225-AD03-4855-8B68-829E6CF1AF98.png]

[image:621B6BCC-F326-4A7D-9906-9CD77BBBD45E-8906-000314F0DC79183C/1A2FBA28-87CF-4AA3-AA69-F709AC10A106.png]

* MyISAM, MEMORY 스토리지 엔진은 ‘3’을 저장하는 순간 중복 PK 오류가 있지만, insert 문을 그대로 실행하면서 먼저 실행된 1,2를 남겨둠
-> Partial Update
* innoDB는 쿼리 중 일부라도 오류가 발생하면 전체를 원 상태로 만들어버림
* 만약, 트랜잭션이 지원되지 않는다면, partial update의 결과로 남은 레코드를 삭제하는 재처리 작업이 필요할 수도..!

```mysql
INSERT INTO tab_a ...;

IF(_is_insert1_succeed) {
	INSERT INTO tab_b ..;

	IF(_is_insert2_succeed) {
		//처리
	} ELSE {
	DELETE FROM tab_a where ...;
	IF(_is_delete_succeed){
		//처리 실 패 및 tab_a, tab_b 복구
	} ELSE {
		//망함. 해결불가능한 심각한 상황
	}
	}
```

##### 주의사항
트랜잭션 또한, *커넥션과 동일하게 꼭 필요한 최소한의 코드에만 적용하자*
= 프로그램 코드에서 *트랜잭션의 범위를 최소화하자*

Ex. 사용자가 게시판에 게시물 작성 후 저장 버튼 클릭했을 때

```
1. 처리 시작
-> 데이터베이스 커넥션 생성
-> 트랜잭션 시작
2. 사용자 로그인 여부 확인
3. 사용자의 글쓰기 내용의 오류 여부 확인
4. 첨부로 업로드된 파일 확인 및 저장
5. 사용자의 입력 내용을 DBMS에 저장
6. 첨부 파일 정보를 DBMS에 저장
7. 저장된 내용 또는 기타 정보를 DBMS에서 조회
8. 게시물 등록에 대한 알림 메일 발송
9. 알림 메일 발송 이력을 DBMS에 저장
<- 트랜잭션 종료 (COMMIT)
<- 데이터베이스 커넥션 반납
10. 처리 완료
```

* 보통은, 데이터베이스 커넥션은 커넥션 풀에서 가져오고 다 사용하면 반납함. 위와 같은 경우, 사실 트랜잭션은 5번부터 시작하고 있음. 2-4를 포함할 필요가 없음 -> 커넥션 풀을 기다려야 하는 경우가 생길 수도
* 메일전송, FTP 등의 네트워크상 원격 서버 통신 작업은 *어떻게 해서든 트랜잭션에서 빼자*.
* 위의 트랜잭션을 나눌 수 있다. 
	* 사용자 입력 정보를 저장하는 5,6
	* 7번은 단순 조회, 굳이 트랜잭션 포함 필요 없음
	* 성격이 다른 9번은 굳이 5,6과 묶일 필요 없음

```
1. 처리 시작
2. 사용자 로그인 여부 확인
3. 사용자의 글쓰기 내용의 오류 여부 확인
4. 첨부로 업로드된 파일 확인 및 저장
-> 데이터베이스 커넥션 생성
-> 트랜잭션 시작
5. 사용자의 입력 내용을 DBMS에 저장
6. 첨부 파일 정보를 DBMS에 저장
<- 트랜잭션 종료 (COMMIT)
7. 저장된 내용 또는 기타 정보를 DBMS에서 조회
8. 게시물 등록에 대한 알림 메일 발송
-> 트랜잭션 시작
9. 알림 메일 발송 이력을 DBMS에 저장
<- 트랜잭션 종료 (COMMIT)
<- 데이터베이스 커넥션 반납
10. 처리 완료
```


#### 4.2 CAP Theorem
[image:FBDFFE3B-89A1-428B-9F40-E47A34FE3015-8906-0003139755EA2ADA/3F4CF19F-76AB-4BB9-AFD7-9CE53832722E.png]


분산 데이터 시스템은 일반적으로 CAP 정리를 따름.

* Consistency: 모든 노드가 가장 최근에, 그리고 성공적으로 저장된 값을 return해야 함. 따라서, 모든 노드는 항상 데이터에 대해 동일한 view를 가지고 있음. *ACID의 Consistency와는 다른 개념

* Availability: non-failing 한 모든 노드는 합리적인 시간 안에 read/write 요청에 대한 non-error response를 return 해야 함.

* Partition-tolerance: 노드 간 임의의 메시지가 누락되거나 노드의 일부가 장애가 나더라도 항상 작동해야함.  ("The system continues to operate despite arbitrary message loss or failure of part of the system")

CAP 정리에 따르면 분산 데이터 시스템은 위의 세 가지 요소를 동시에 보장할 수 없고, 실질적으로는 availability나 consistency 중 택하게 됨.

> 위의 정의는 2000년 Brewer가 정리를 발표할 때의 정의이지만, 사실 2002년 Gilbert,Lynch에 의한 재정의에 따르면 "The network will be allowed to lose arbitrarily many messages sent from one node to another"이다. 사실, C와 A는 분산 시스템의 특성이지만 P는 분산 시스템이 돌아가는 네트워크의 특징이다. P의 의미를 따져보면 '네트워크가 임의의 메시지를 손실할 수 있냐', 즉 네트워크가 가끔 장애가 날 수도 있다는 것을 인정하느냐에 관한 문제이고, 절대 장애가 없는 네트워크는 존재할 수 없다. 따라서 사실 P는 어쩔 수 없이 선택되어야 하는 것이다. 따라서 결국 C,A 중 하나를 선택해야 하는 것인데, 이 둘은 대칭적이라고 보기도 어렵다. Cassandra는  네트워크 장애 상황이든 정상 상황이든 C를 포기하도록 되어있고, HBase의 경우에는 장애상황일 때만 A를 포기하고 정상일 때는 A를 만족한다. CAP Theorem은 네트워크가 장애 상황일 때만 대칭성이 있고 정상일 때에는 그렇지 않다. 결국, CAP Theorem은 "분산시스템에서 네트워크 장애상황일 때 일관성과 가용성 중 많아도 하나만 선택할 수 있다" 는 것으로, 정상 상황에 대한 특성을 설명하지 못한다. 따라서, PACEL이 등장하게 되었다. 비슷한 논의로, 사실 RDBMS는 분산 시스템이 아니기 때문에 CA로 분류하기 어렵다. 

* CP 시스템
* AP 시스템

#### 4.3. BASE Systems

분산 시스템에서의 고가용성을 위한 특성

* Basically-Available: 고가용성을 유지해야 함. 데이터 시스템은 만약 응답이 오래되더라도(stale) 항상 요청에 대해 응답할 수 있어야 한다.
* Soft-state: 시스템의 상태가 별도의 입력을 받지 않아도 변할 수 있다. 따라서, 시스템은 항상 eventual consistency의 상태에 도달할 수 있도록 soft state 상태이다. 
* Eventual consistency: 시스템이 어느 시점부터 입력 받기를 멈춘다면, 시스템은 결국 consistent한 상태에 도달할 것이다.데이터의 변경사항은 결국 모든 노드에게 전파될 것이고, 모든 노드는 동일한 데이터를 바라볼 것이다.

consistency에 대한 강한 제약을 완화해서 고가용성을 확보함

### 5. Distributed Commit Protocol

local transaction에서는 하나의 DB만을 고려하기 때문에 DB 차원에서 직접적인 transaction 컨트롤이 가능하며, 어플리케이션 레벨에서도 적절한 API를 사용해 컨트롤이 가능하다.

하지만 분산 시스템에서는 다수의 리소스가 관여되기 때문에 DB 차원의 transaction 관리가 어렵다. 따라서, 분산 시스템에서는 *독립적인 리소스(DB) 들과 이들이 서로 협력하기 위한 transaction coordinator*가 필요하다.

#### 5.1. Two-phase Commit
[image:AAD6D2B9-EE20-42B9-841B-41B7E68ED7DA-8906-000313A31C04721D/675AFB94-34AD-4723-B31D-BBB37AEBC34C.png]

분산 시스템의 transaction의 ACID를 보장하기 위한, 가장 보편적인 방법

* application이 여러 노드에 데이터를 write하면서 transaction 시작
* Prepare Phase(phase1) : transaction coordinator가 모든 참여자에게 커밋을 준비해달라고 요청. 참여자들은 긍정/부정으로 응답할 수 있음.
* Commit Phase(phase2) : 이전 단계에서 모두에게 yes를 받으면 모든 참여자에게 commit을 요청. 만약 하나라도 no를 받으면(or timeout, failure..) abort ← 해당 결정을 transaction coordinator log(디스크)에 기록, 참여자가 다운되거나 timeout이 난다거나 하는 경우 얼마나 retry를 하던 결정이 번복되지 않도록 해야 함.  (참여자가 다운되는 경우, 회복되었을 때 commit)
* 즉, 장애가 나더라도 coordinator의 결정을 강제하는 방법으로 ACID 보장

[image:10A884D6-61A9-4AF3-B86A-A88E2B989016-8906-000313A9F0752231/2C0EA185-AF10-434D-A5F3-E6B5AA380B57.png]


* 단, prepare phase에서 참여자들이 yes를 보낸 이후 coordinator가 다운되는 경우, 회복될 때 까지 기다리는 수밖에 없음. 회복 후 로그를 확인하여 트랜잭션 commit/abort 수행

ex)
[image:D23E174B-0D7E-49DF-B25E-858A4CB00649-8906-000313ADCF794C1E/43593629-F524-44B7-908D-AC83EDAACF4C.png]

[image:75C4680D-9EED-4B96-9DF7-1304231231AF-8906-000313B8122F1081/B8D36D01-742F-4586-B97A-557C8E4D02A9.png]

(출처: https://blog.daum.net/ossogood/8435405)

#### 5.2. Three-Phase Commit
[image:B79B7315-1F01-43CA-8AA1-72D944A563F7-8906-000313BB214991C0/AA46496E-57E8-4DB7-A9CB-B5A46138E7C4.png]


위의 단점을 극복하기 위해, commit phase를 pre-commit phase, commit-phase로 분리

pre-commit phase를 통해 참여자가 실패하는 경우 / coordinator와 참여자가 실패하는 경우의 recover를 가능하게 한다.

recovery coordinator가 pre-commit phase를 사용해 commit 해야 할 지, abort 해야 할 지 안전하게 결정

이러한 프로토콜들은 ACID를 보장해주지만, 블로킹 프로토콜 이라는 문제가 존재

### 6. Industry Specifications

업체들은 two-phase commit과 같은 분산 트랜잭션 프로토콜을 독립적으록 구현할 수는 있으나, 다른 여러 업체들과 협업하는 경우 상호 운용성이 어려워진다. 특히, 트랜잭션의 메시지 큐와 같은 이질적인(heterogeneous) 리소스들을 다룰 때 그 복잡도가 훨씬 커진다. 

따라서, 분산 트랜잭션의 표준 스펙을 정의하고자 하는 여러 협업이 있었다.

#### 6.1. X/Open DTP Model

[image:33B447B2-8505-4827-85C7-8BD504FB82F4-8906-000313BFD0BC2835/C7345AA1-73A9-4541-9F9D-43C7099D8E1D.png]


X/Open 컨소시엄에서 릴리즈되었는데, 상이한 컴포넌트들을 포함한 글로벌 transaction에 atomicity를 제공하기 위한 목적을 가짐. two-phase commit protocol 기반으로 data integrity를 보장하며, 표준화된 캄포넌트와 인터페이스를 가진다. 아래는 XA 스펙의 일부분이며 실제로는 더 다양한 컴포넌트를 가진다.

* Application Program: AP는 트랜잭션을 정의하고, 트랜잭션의 범위 안에서 리소스에 접근하는 책임을 가짐. 글로벌 트랜잭션의 시작과 끝을 정의하기 위해 transaction manager를 사용
* Transaction Manager: TM은 분산 트랜잭션에서의 작업 단위인 global transaction을 관리하는 책임을 가지며, commit/rollback 의사결정을 조율하고 failure recovery를 담당한다.
* Resource Manager: RM은 DB와 같이 공유 리소스의 특정 부분을 관리하는 책임을 가지며, TM과 협력하여 글로벌 transaction의 일부인 transaction branch를 조절한다.

#### 6.2 OMG OTS Model

OMG(Object Management Group, 분산 객체에 대한 표준을 정의하기 위해 만들어진 컨소시엄)에서 제안된 Object Management Architecture (OMA)의 일부. Object Transaction Service라는 의미

* 객체지향의 관점에서 분산형 어플리케이션의 통신 인프라 모델
* CORBA 어플리케이션에서 distributed two-phase commit transaction 사용을 가능하게 함

> Coba? 
> OMG에서 정의한 규격으로서 소프트웨어 컴포넌트들을 언어와 사용환경에 대한 제약이 없는 통합을 위한 표준을 지칭. 분산 컴퓨팅과 객체지향 기술을 하나로 합친 표준 아키텍쳐
[image:6A05BCAC-F8FC-45AA-AC0F-FDE9DE7F3C50-8906-000313C9312535EA/70A9E461-2975-47FC-B6A3-503DFCD1688E.png]


* Transactional Server: 트랜잭션과 관련된 하나 이상의 객체를 보관
* Transactional Client: transactional object의 메서드를 호출하는 프로그램
* Recoverable Server: 트랜잭션의 commit, rollback에 영향을 받는 transactional object인 하나 이상의 recoverable object를 보관
* Transactional Object: transactional context에서 호출될 수 있는 메서드가 있는 CORBA 객체

### 7. Long-Running Transactions

ACID를 보장하는 프로토콜들은 blocking 문제가 발생할 수 밖에 없음. ex) MSA에서 2P-commit을 적용한다면, 한 서비스가 장애났을 경우 전체 서비스가 락이 걸려야 함

따라서, 짧은 시간의 트랜잭션에서는 잘 작동하지만, long-running business transaction에서는 효과적이지 못함.

* applcation의 scalability 저하
* resource locking을 사용하는 전통적인 기술들은, 느슨하게 결합된 비동기 환경의 business transaction에 어울리지 않음

#### 7.1.Saga Interaction Pattern

SAGA 패턴은 하나의 long-running process를 여러개의 작은 business action과 interaction으로 쪼개고, 메시지와 타임아웃 기반으로 프로세스를 조절함

*성공*
[image:DC63BCFF-05EC-4ED2-94E7-9021C5AD5C61-8906-000313CEE4E579AE/16F8E165-8E72-453F-BE98-67968855B7A8.png]


*실패*
[image:30154823-144E-42E3-9D18-B0049E6AE90E-8906-000313DD5A87FA33/CAEC5B6A-97A3-43F3-B40A-21697D8029D5.png]

* 트랜잭션의 관리주체가 DBMS가 아닌 Application에 존재
* 각 App 하위에 존재하는 DB는 Local 트랜잭션 처리만 담당합니다.
* rollback 처리 (보상 트랜잭션)을 Application에서 구현해야함
* Eventually Consistency 보장
* 다양한 DBMS 사용가능

### 8. Advanced Consensus Protocols

[image:AAA46BF2-5765-4AEC-93F5-AB980E33966B-8906-000313E3E07CD174/C85E351A-3AE9-471F-959B-C06FDF791923.png]


transaction을 commit 할 것이냐, roll-back 할 것이냐는 결국 분산 시스템에 있어서 consensus-problem의 일부

consensus?

> Consensus refers to a process for distributed processes to agree on some state or decision.

분산 시스템에서 프로세스들이 어떤 상태, 혹은 어떤 결정을 내리는데 필요한 절차

consensus problem을 해결하기 위해서는 consensus protocol이 필요함

* eventual termination, data integrity, 노드(프로세스)간의 합의가 필요함
* consensus protocol의 평가기준은 running time & message complexity

ex) Two-phase commit protocol

* low message complexity
* low overall latency
* 하지만 blocking 존재

#### 8.1 Paxos

1989 Leslie Lamport가 제안한, 가장 유명한 consensus protocol, asynchronous & unreliable process에서 consensus problem을 해결할 수 있음.

다양한 버전이 있으나, 가장 간단한 버전을 살펴보고자 함. 'Paxos made simple'

* Proposer, Acceptor, (Learner)
* 2개의 페이즈로 구성된 라운드가 여러번 반복됨

* Prepare(Phase 1A): Proposer가 유니크한 인덱스를 가진 메시지를 acceptor들에게 보낸다.
* Promise(Phase 1B): Acceptor가 Proposer들의 메시지를 받아 인덱스를 확인하고, 만약 인덱스가 지금까지 받은 메시지에서 가장 큰 숫자면 Proposer에게 Promise를 보낸다.*
* Accept(Phase 2A): Proposer은 정족수의 Acceptors로부터 많은 Promise를 받을 수 있으며, 선택된 값과 함께 정족수의 Acceptors에게 message를 보낸다.
* Accepted(Phase 2B): Acceptor들은 accept message를 받고, 만약 그 acceptor가 받은 message보다 더 큰 인덱스에 promise를 보내지 않았다면, accept한다.

다수의 proposer가 서로 상충되는 메세지를 보낼 수도 있으며, acceptors는 복수의 proposal을 accept 할 수 있다. 

하나의 round에서는 아무 proposal도 채택되지 않을 수 있지만, Paxos 내에서는 결국 하나의 값에 합의할 수 있도록 보장된다.


























