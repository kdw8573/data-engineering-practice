- [Kafka 개념](#kafka-개념)
  - [Kafka 이전의 데이터 처리](#kafka-이전의-데이터-처리)
  - [아파치 카프카(Apache Kafka)란?](#아파치-카프카apache-kafka란)
  - [Kafka 특징](#kafka-특징)
  - [Core Components](#core-components)
  - [고가용성(High Availability)](#고가용성high-availability)
  - [Controller](#controller)
  - [Producer 메시지 전달 과정](#producer-메시지-전달-과정)
  - [Producer 메시지 저장 방식](#producer-메시지-저장-방식)
  - [Consumer 특징](#consumer-특징)
  - [Segment File](#segment-file)
  - [메시지 보증 전략](#메시지-보증-전략)
  - [Kafka Connect](#kafka-connect)
  - [Schema Registry](#schema-registry)

# Kafka 개념

## Kafka 이전의 데이터 처리
* 일반적으로 데이터를 먼저 저장했다가 나중에 임의의 시간 간격으로 처리하는 배치 작업으로 처리
* 실시간 처리가 아니여서 즉각적인 비즈니즈 대응이 어려움.(ex. 통신사에 실시간 요금 제공...)
* 그래서 이벤트 스트리밍 필요성 발생. 이벤트가 생성되는 대로 이벤트의 스트림을 지속적으로 처리하는 프로세스의 요구.

## 아파치 카프카(Apache Kafka)란?
* 스트리밍 이벤트 데이터 또는 일반 데이터를 수집, 처리, 저장하는 데 널리 사용되는 이벤트 스트리밍 플랫폼.
* Publish-Subscribe(Pub-Sub) 모델을 통해 구현한 분산 메시징 시스템(로그 데이터, 이벤트 메시지 등의 데이터를 처리하는 시스템). 
* Pub-Sub 모델은 데이터를 만들어내는 프로듀서, 소비하는 컨슈머. 둘 사이에서 중재자 역할을 하는 브로커로 구성된 느슨한 결합(Loosely Coupled)의 시스템.
* Producer는 메시지를 생성한 뒤 broker에게 전달. Broker는 메시지를 topic별로 분류하여 쌓아둠. Consumer는 Broker에게서 메시지를 가져감.
* 링크드인에서 처음 개발 -> 2011년 오픈소스로 공개 -> 2012년 10월 아파치 인큐베이터 종료 -> 2014년 일부 엔지니어들이 Confluent 회사 창립.

## Kafka 특징
1. 여러 Producer가 동시에 메시지를 전송할 수 있고, 여러 Consumer에서 동시에 메시지를 읽을 수 있다.
2. 프로듀서 전달한 메시지를 브로커의 동작에 영향을 주지 않고 처리 속도 및 장애 복구(fail over)를 유지할 수 있게 하기 위해 일정 기간 동안 파일 형태로 저장.
3. 처리 속도가 저하되면 Consumer 또는 Producer를 조절하여 처리량을 늘릴 수 있다. 
4. Consumer가 프로듀서의 메시지 생성속도를 따라가지 못할 때 컨슈머를 그룹으로 묶어 다른 컨슈머를 추가함으로써 프로듀서에서 보내는 속도와 읽는 속도의 균형을 맞출 수 있다. 
 
## Core Components
![kafka components](https://open.oss.navercorp.com/storage/user/1981/files/55e049ae-8f3f-4ec8-bb0b-536158b7c51a)
* Producer: 메시지를 발생시키고, 카프카 cluster에 적재.
* Consumer: 카프카 cluster에서 메시지를 구독하고 가져옴.
* Zookeeper: 카프카 클러스터 정보 및 분산처리 관리 등 메타데이터 저장(broker id, controller id...)
* Kafka Cluster: 카프카 broker들의 모임. 확장성과 고가용성을 위해 broker들이 cluster(여러 대의 컴퓨터들이 연결되어 하나의 시스템처럼 동작하는 컴퓨터들의 집합)로 구성
* Broker: Kafka 서버를 의미.
> * Topic: 카프카에 전달되는 메시지 스트림의 추상화된 개념. 프로듀서는 메시지를 특정 토픽에 발행한다. 토픽은 카프카 클러스터에서 여러개를 생성할 수 있으며, 하나의 토픽은 1개 이상의 파티션으로 구성. 
> * Partition: 토픽 당 데이터를 분산 처리하는 단위. 토픽 안에 파티션을 나누어 데이터를 분산처리한다. 그래서 파티션 수 만큼 Consumer를 연결 할 수 있다. 파티션으로 메시지가 들어오면 append-only 특성이여서 추가만 가능. 파티션의 수를 나중에 증가시키는 것은 가능하지만, 줄이는 것은 불가능하므로 처음 토픽을 생성할 때 신중하게 파티션 수를 결정해야 된다.
> * Offset: 파티션 내의 각 메시지들의 위치. 프로듀서로부터 메시지가 들어오면 파티션의 가장 끝에 붙게되며, 컨슈머는 offset을 기준으로 어디까지 읽었는지 판단이 가능.  
> ![kafka topic](https://open.oss.navercorp.com/storage/user/1981/files/35a01402-e3c4-4aeb-9a0a-95d131eca875)

## 고가용성(High Availability)
* 서버, 프로그램 등이 오랜 시간 지속적으로 정상 운영이 가능한 성질을 의미.
* 카프카에서는 고가용성을 제공하기 위해 토픽의 replication factor 설정 값을 이용해, 토픽의 파티션을 각각의 브로커로 설정 값만큼 복사.
* 각 파티션에 대해 N개의 복사본은 replica가 되며, 1개의 replica는 Leader가 되고, 나머지 N-1의 replica는 Follower가 된다.
* Producer와 Consumer의 read, write 요청은 Leader replica에게만 전달되며, Follower는 리더가 속한 브로커에게서 메시지를 복사.
* 리더의 변경사항을 잘 따라가는 Follower는 ISR(In-Sync Replica)가 되며 만약 Leader replica가 있는 브로커에 장애가 생길 경우, ISR에 속한 replica 중에 새로운 Leader로 선정.
* Broker의 로드를 줄이기 위해 가급적 브로커마다 동일한 Partition leader 설정.
* 복제 개수만큼 저장용량이 증가한다는 단점이 존재하지만, 데이터를 안전하게 사용할수 있다는 강력한 장점.
> ![kafka replication](https://open.oss.navercorp.com/storage/user/1981/files/ac5fe49a-ddfa-4a83-9e2d-ac5056bdb543)

## Controller 
* 카프카 브로커 중 하나로, 다른 브로커들의 생존 여부를 확인. 
* 주키퍼의 /controller 노드에 임시노드를 가정 먼저 생성한 브로커가 controller로 선출되며, 만약 컨트롤러 브로커가 중단되면 /controller 에서 임시노드가 삭제.
* 그럼 다른 브로커에서 /controller에 다시 임시노드 생성을 시도해 새로운 controller로 선출.
* 만약 임의의 브로커가 중단되면, 컨트롤러는 다른 브로커에 있던 팔로워 파티션 중 하나를 골라 리더 파티션으로 선출되고 변경된 리더 파티션 정보를 주키퍼에 저장.

![kafka controller](https://open.oss.navercorp.com/storage/user/1981/files/9b70343f-55e2-4901-bc4f-b2139c66c927)

## Producer 메시지 전달 과정
1. 직렬화: 직렬화란 클래스 내부에 있는 데이터, 변수 등의 값들을 byte 형식의 데이터로 만드는 것. 프로듀서는 전달된 메시지의 key와 value를 지정된 Serializer로 byte 데이터로 변환.
2. 파티셔닝: 직렬화된 메시지는 partitioner를 통해 토픽의 어느 파티션에 저장될지 결정. 별도의 설정이 없으면 Round-Robin 방식으로 메시지가 저장되며, 특정 파티션에 메시지를 저장하도록 지정할 수도 있다.
3. 압축: 메시지 압축 설정이 있다면, 설정된 포맷에 맞춰 메시지를 압축. 압축된 메시지는 브로커로 빠르게 전달할 수 있으며, 브로커 내부에서 빠르게 복제 가능.
> ![kafka compression](https://open.oss.navercorp.com/storage/user/1981/files/aacb7a42-8949-40e6-9773-5efc3d5c2036)
4. 배치: 프로듀서는 메시지를 네트워크를 통해 리더 파티션으로 전송. 그러나 네트워크 전송은 느리기 때문에 메시지가 올 때마다 전달하는 것은 비효율적. 그래서 지정된 만큼 메시지를 저장했다가 한 번에 브로커에 전달. 프로듀서 내부의 Record Accumulator(RA)가 담당하여 처리하며 메시지들을 레코드 배치(Record Batch) 형태로 묶어 큐에 저장. 
> * batch size: producer 옵션으로 배치로 보내려는 데이터 양을 조절.
> * linger.ms: 최대 전송 대기 시간 설정 옵션. 값을 조절하여 batch.size가 꽉 차도록 설정하면 서버의 부하를 줄일 수 있다. 대신 값이 높을수록 처리량은 증가하지만, 그만큼 저장되었다가 전송되므로 지연율이 높아짐.
5. 전달: 
> * Sender를 통해 레코드 배치를 브로커로 전송되며 설정 값에 따라 브로커 응답을 기다리거나 기다리지 않음. 
> * 응답을 기다리는 경우, 메시지 전송 성공 여부를 응답으로 받는다. 이때, 브로커에서 메시지 전송이 실패한 경우에는 retries 값에 따라 재시도. 
> * 재시도 횟수를 초과한 경우에는 예외를 발생. 성공한 경우에는 메타데이터를 반환. 
> * 메타데이터에는 메시지가 저장된 토픽, 파티션, 오프셋, 타임스탬프 정보를 가지고 있음.

## Producer 메시지 저장 방식
* acks = 0: 리더 파티션에 데이터를 전달하고 응답값을 받지 않음. 리더 파티션에 데이터가 정상적으로 전송되었는지, 팔로워 파티션에 복제가 되었는지 확인 불가. 따라서 데이터 전달 속도는 빠르지만, 브로커에 장애가 생겼을 경우 데이터 유실 가능성이 있음.
* acks = 1: 리더 파티션에 데이터를 전달하고 응답값을 받음. 그러나 팔로워 파티션에 정상적으로 복제 되었는지는 확인 안함. 따라서 리더 파티션이 있는 브로커에 데이터를 받은 즉시 장애가 생겼을 경우 데이터 유실 가능성 있음.
* acks = all: 리더 파티션에 데이터를 전달하고, 팔로워 파티션에도 데이터가 정상 복제되었는지 확인. 데이터 유실 가능성은 적지만, 확인 과정이 길어져 데이터 전달 속도가 느림.

## Consumer 특징
* polling 구조: 컨슈머는 자신이 원하는 만큼만 브로커에게 메시지 요청. 각 컨슈머는 자신의 환경에 맞게 메시지를 읽을 수 있어 최적화 가능.
* 단일 Topic, Multi Consuming: 하나의 토픽에 서로 다른 컨슈머가 동시에 구독 가능. 컨슈머가 메시지를 읽을 때 브로커에서 메시지가 삭제되는 것이 아니고 각 컨슈머가 파티션의 어느 오프셋까지 읽었는지를 '__consumer_offset' 토픽에 저장. 따라서 컨슈머 오프셋 토픽에 저장된 정보를 통해 어디서부터 메시지를 읽어야되는지 알 수 있음. 즉, consumer 상태와 관련없이 안정적인 메시지 구독이 가능.
* Consumer Group: 
> * 하나 이상의 컨슈머가 컨슈머 그룹을 구성하여 하나의 토픽을 구독할 수 있음. 
> * 컨슈머 그룹 내의 컨슈머는 파티션의 소유권을 나눠 가짐. 즉, 같은 컨슈머 그룹에 속한 컨슈머들이 동시에 동일한 파티션에서 메시지를 읽어갈 수 없다. 
> * 컨슈머 그룹은 토픽이 특정 파티션으로부터 데이터를 가져가고 이 파티션의 어느 레코드까지 가져갔는지 확인하기 위해 오프셋을 커밋. 커밋한 오프셋은 __consumer_offsets 토픽에 저장.
> * 파티션 수와 컨슈머는 일대일로 매핑 해야하는 것은 아니지만, 파티션 수보다 컨슈머 수가 많게 구현되는 것은 바람직한 구성은 아님. 컨슈머 수가 파티션 수보다 많다면 더 많은 수의 컨슈머들은 그냥 대기 상태로 존재하기 때문. 
> * 컨슈머 그룹 내에서 컨슈머 수가 변경되면 파티션 소유권이 재조정되는 리밸런싱이 발생.  
> * 리밸런싱을 통해 컨슈머 그룹 내의 컨슈머들이 파티션을 고르게 할당받아 소비할 수 있다.
> * 브로커가 추가/제거 되는 경우 전체 컨슈머 그룹들에서 리밸런싱이 발생.
> ![kafka consumer group](https://open.oss.navercorp.com/storage/user/1981/files/871b2dee-e697-4880-9549-28c852919c82)

## Segment File
* 메시지는 topic안에 partition별로 로그를 쓰는데, 이 때 로그를 쓰고 보관하는 단위가 segment.
> ![kafka segment](https://open.oss.navercorp.com/storage/user/1981/files/28274b43-96f0-480b-b96a-b7dae8ab74a8)
* kafka 폴더의 server.properties파일에서 logs.dir 값을 통해 segment 파일이 저장되는 위치를 알 수 있다.
* server.properties에서 로그와 관련된 옵션을 지정할 수 있다.
> * log.segment.bytes : 바이트 단위로 최대 세그먼트 크기를 정의 (default 1GB)
> * log.retention.{ms, minutes, hours} : 로그 세그먼트를 보유할 시간을 정의 (default: 7일)
> * log.roll.hours: 새로운 세그먼트 파일이 생성된 이후 크기한도에 도달하지 않았더라도 다음 파일로 넘어갈 시간 주기(default 7일)
* 메시지를 파일에 저장함으로 인해 컨슈머쪽에 장애가 생겨 메시지를 바로 소비하지 못 해도, 카프카가 메시지를 보존하고 있는 기간 내에는 언제든 읽어 갈 수 있다.
* 일반적으로 파일 시스템을 사용하면 속도가 느리다는 단점이 있다. 그러나 카프카는 page cache와 zero copy를 통해 최적화.
* page cache는 메인 메모리의 남는 공간에 파일의 내용을 담는 것을 의미. 이를 통해 consumer에서 읽기 요청이 왔을 때 disk에 접근하지 않고, 메모리에 적재되어 있는 데이터를 바로 전송.
* zero copy는 카프카 application에 데이터를 올리지 않고, 디스크에서 있는 세그먼트 파일로부터 메시지를 읽음과 동시에 네트워크에 전송하는 것을 의미. 따라서 disk와 네트워크 사이에 전송 속도가 올라간다.
> ![zero_copy_2](https://open.oss.navercorp.com/storage/user/1981/files/6b16ad03-c7fe-4ddd-8530-244cc3547e6e)

## 메시지 보증 전략
* at-most-once: 실패나 타임아웃 등이 발생하면 메시지를 버릴 수 있다. 데이터가 일부 누락되더라도 영향이 없는 대량처리나 짧은 주기의 전송 서비스에 유용할 수 있다.
* exactly-once: 메시지가 정확하게 한 번만 전달되는 것을 보장한다. 손실이나 중복 없이, 순서대로 메시지를 전송하는 것은 구현 난이도가 높고 비용이 많이 든다.
* at-least-once: 메시지가 최소 1번 이상 전달되는 것을 보장한다. 실패나 타임아웃 등이 발생하면 메시지를 다시 전송하며, 이 경우엔 동일한 메시지가 중복으로 처리될 수 있다.
> * 카프카는 ack를 1이나 all로 설정함으로써 at-least-once를 보장할 수 있다. 
> * 만약 네트워크 상에서 ack가 소실 또는 지연되어 수신 실패할 경우, Producer는 메시지 전송이 실패했다고 판단하여 재전송한다.
> * Consumer가 메시지를 읽은 후 offset을 갱신하기 전에 장애가 발생할 경우, Consumer는 재시작되었을 때 갱신되지 않은 offset을 기준으로 메시지를 읽어와 동일한 메시지를 중복으로 읽어올 수 있다. 

## Kafka Connect
* 반복적인 파이프라인 작업시 매번 프로듀서,컨슈머를 개발하고 배포 하는것은 비효율적. 카프카 커넥트를 이용하면 특정한 작업을 템플릿으로 만든 커넥터를 실행함으로써 반복작업을 줄일 수 있다. 즉, Kafka Connect는 Connector를 이용하여 데이터소스와 kafka를 연결해준다.
* 유저가 Connect에 Connector 생성 명령을 내리면 커넥트는 내부적으로 커넥터와 태스크를 생성. 커넥터는 프로듀서 역할을 하는 소스커넥터와 컨슈머 역할을 하는 싱크커넥터가 존재.
> * Source Connector : 데이터 소스의 데이터를 카프카 토픽으로 Publish 하는 커넥터.
> * Sink Connector : 카프카 토픽의 데이터를 Subscribe해서 Target System에 반영하는 커넥터.
* connect에 옵션으로 컨버터와 트랜스폼 기능을 넣을 수 있다. 
> * 컨버터: 데이터의 serialization, deserialization를 처리. JsonConverter, StringConverter, ByteArrayConverter 등이 존재.
> * 트랜스폼: 데이터 처리 시 각 메시지 단위로 데이터를 간단하게 변환하기 위한 용도로 사용.
* 오픈소스 커넥터는 직접 커넥터를 만들 필요가 없이 파일을 다운로드 하여 사용할 수 있다는 장점. HDFS, AWS S3, JDBC 커넥터 등이 존재.
![kafka connect](https://open.oss.navercorp.com/storage/user/1981/files/41071039-2e80-4517-b754-26e216d660a2)

## Schema Registry
* kafka는 pub-sub 모델로 느슨한 결합이라고 했지만, 실제로는 메시지에 대해 producer와 consumer 간에 의존성이 강하다.
* 프로듀서는 직렬화하여 메시지를 발행하고, 컨슈머는 역직렬화하여 메시지를 구독하므로 메시지 구조(스키마)에 따라 직렬화/역직렬화 클래스를 필요로 한다.
* 즉, 바이트 배열이 역직렬화 로직과 맞지 않다면(스키마가 변경) 정상적으로 메시지를 처리할 수 없다.
* 스키마 레지스트리는 이러한 의존성을 없애기 위해 고안됨. RESTful 인터페이스를 사용하여 스키마(Schema)를 관리하거나 조회하는 기능을 제공.
* producer에서 데이터의 스키마 정보는 schema registry로 보내고, data는 kafka broker로 보냄.
* consumer는 Kafka 로부터 데이터를 받는데 이 데이터에는 스키마 ID가 포함되어 있고, Schema Registry에서 스키마정보를 탐색하여 가져와서 역직렬화하여 사용.
* 스키마 관리 전략
> * Backward: 새로운 스키마로 이전 데이터를 읽는것이 가능한것을 의미. 새로 추가되는 필드에 default value를 지정할때에만 스키마 등록이 허용. 기본적으로 Backward로 동작.
> * Forward: 이전 스키마에서 새로운 데이터를 읽는것이 가능한것을 의미. 새로운 스키마에서 특정 필드가 삭제된다면, 해당 필드는 이전 스키마에서 default value를 가지고 있어야 함.
> * Full: Backward 와 Forward 을 모두 만족함
> * None: 스키마 호환성을 체크하지 않음.
![kafka schema](https://open.oss.navercorp.com/storage/user/1981/files/11bf7921-15af-4b7e-905c-f84b972216fb)
