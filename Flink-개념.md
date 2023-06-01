- [Flink 개념](#flink-개념)
  - [Stream Processing](#stream-processing)
  - [Flink 특징](#flink-특징)
  - [Flink 프로그램 실행 순서](#flink-프로그램-실행-순서)
  - [Flink abstraction Level](#flink-abstraction-level)
  - [Window](#window)
  - [DataStream / DataSet](#datastream--dataset)
  - [Flink Streaming DataFlow](#flink-streaming-dataflow)
  - [Flink 구조](#flink-구조)
  - [Time](#time)
  - [Watermarks](#watermarks)
  - [CheckPoint](#checkpoint)

# Flink 개념

## Stream Processing
* 스트림 프로세싱은 시작과 끝이 없는 Unbounded 데이터를 입력으로 받아 끊임없이 데이터를 처리해 결과를 만들어내는 행위.
* 여러개의 Data Source로부터 연속적으로 생성되는 데이터를 레코드나 정의된 단위에 따라 순차적으로 처리한다. 이때 처리에 의미는 단순 수집, 합계·평균 같은 집계, 데이터 형식 변환, 다른 데이터 소스와 결합 등을 가리킴.
* 구현방식에 따라 native 방식과 batch 방식이 존재.
> * native streaming: 모든 records(events)를 스트리밍 시스템에 도착하자마자 처리. 즉시 처리되므로 프레임워크가 최소한의 latency(대기시간)를 가짐. 그러나 장애가 생겼을 때 fault tolerance 보장이 어렵다. 지속적으로 유입되는 데이터와 계속 실행되는 Job으로 인해 실패한 부분의 연산을 재실행 하는 것이 쉽지 않기 때문. ex) storm, flink
> * micro-batching : 유입되는 records(events)를 짧은 주기(default: few seconds)로 batch 가능한 단위로 묶어서 처리. 데이터 처리에 실패했을 때는 연산에 실패한 부분을 재실행함으로써 fault tolerance를 보장하기기가 쉽다. 대신 record를 모아서 처리하기 때문에 latency가 높다. ex) spark streaming

## Flink 특징
* 베를린 TU대학교에서 시작된 아파치 프로젝트. 2011년 첫 릴리즈.
* flink는 분산 한경에서 네이티브 형식으로 스트림 프로세싱을 처리하는 framework. batch 처리를 위한 API도 제공.
* JVM 기반으로 실행. JVM Garbage Collector 대신에 커스텀 메모리 매니저를 구현하여 안정적인 메모리 사용. 
* Exactly-once 보장. native streaming 이므로 low-latency.

## Flink 프로그램 실행 순서
1. Obtain an execution environment: ExecutionEnvironment(실행환경)를 생성해 DataStream, DataSet을 만들기 위한 준비
2. Load/Create the initial data: Data source를 생성해 input 데이터를 가져옴
3. Specify transformations on this data: 데이터를 변환 및 가공
4. Specify where to put the results of your computations: 계산된 결과를 저장하거나 활용
5. Trigger the program execution: 주기적으로 프로그램 실행

## Flink abstraction Level
* flink는 streaming이나 batch data를 처리하기 위한 여러 수준의 api를 제공함.
* Stateful Stream Processing : 사용자가 직접 state, time 등을 관리할 수 있는 low-level.
* DataStream / DataSet API : 핵심적으로 가장 많이 사용하는 Core API. 데이터 처리를 위해 transformation, join, window 등을 개념을 제공.
* Table API : Library로 제공되는 Table API. 기존의 dataset이나 datastream, data source로 테이블을 만들 수 있다. 테이블 api를 통해 join, select, filter 등의 operation을 쓸 수 있다.  
* SQL : select, join, aggregate 등의 SQL를 사용할 수 있는 High-level Language를 지원.

## Window
* Streaming data는 unbounded data이기 때문에 각 element를 개별적으로 처리 하는 연산이라면 문제 없다.
* 그러나 집계 처리를 할 때는 처음과 끝을 모르기 때문에 계산이 불가능. 그래서 unbounded data를 집계 처리(ex.평균) 등의 가공 처리를 위해 특정한 룰에 따라 일정 데이터를 모아 처리하는 개념.
* data를 window로 할당할 때 4가지 방식을 지원
> * Tumbling: 고정된 단위 시간에 따라 윈도우를 나누어 데이터를 중복 없이 처리. 지정된 기간을 윈도우로 만들어 그 안에서 데이터 처리를 원할 때 사용.
> * Sliding: 특정 시간을 기준으로 앞 뒤를 일정한 간격으로 한 윈도우를 가짐. 중복 데이터가 허용된다. 특정 기간 내 평균값 처리 등을 목적으로 할 때 사용.
> * Session: 일정 기간동안 반응이 없는 경우(session gap) 그 전 window시점부터 마지막으로 데이터가 들어온 시점까지 하나의 window로 처리. 그래서 윈도우 크기가 일정하지 않고, 데이터 양도 다르다.
> * Global : 하나의 윈도우로 모든 데이터 처리. 따라서 trigger(가져올 데이터에 대한 정의) 와 evictor(처리할 데이터에 대한 정의) 를 설정해야 된다. 
> * ex) stream.windowAll(GlobalWindows.create()).trigger(CountTrigger.of(3)) // 입력 데이터 3개마다 처리 .evictor(CountEvictor.of(5)) // 5개의 데이터를 처리

## DataStream / DataSet
* flink는 data가 streaming과 batch 방식에 대해 둘 다 지원. 대신 사용하는 API가 다름.
* DataStream API는 streaming 방식으로 계속 들어오는 데이터를 처리.
* DataSet API는 batch 방식으로 들어오는 데이터를 처리. flink 1.12 version부터 deprecated 되었으며 table API로 대체되었다.

## Flink Streaming DataFlow
* 데이터는 Source operator -> Transformation operator -> Sink operator 순서로 처리.
* Source: 데이터 입력을 정의하는 단계. 로그, 센서 등에서 발생하는 데이터를 실시간 이벤트 스트림이나, DB, file 등에서 수신 받음.
* Transformation: 데이터를 가공하는 단계. 자바에서 제공하는 map, filter, reduce 등의 기능들을 쓸 수 있다. 또한, 트랜스폼 단계에서 윈도우 기능을 제공. 
* Sink: 처리한 스트림을 출력, 저장하는 단계. flink는 데이터를 chain 방식으로 단계별로 변환 시키는 구조를 가지는데, 이 때 계산의 결과 값이 필요할 때까지 계산을 늦추는 Lazy Evaluation을 채택. 그래서 sink 단계가 없다면 데이터 처리를 하지 않는다. 즉, Sink가 실행되는 순간에 정의했던 transformation들이 실행된다.

## Flink 구조
* Flink는 master Process(Job Manager)와 Worker Process(Task Manager)의 두 종류 process가 존재.
* client는 program 실행의 부분이 아니고, JobManager에 dataflow를 준비해서 보내주는 역할.
* JobManager: task들의 schedule을 관리하고, task의 성공과 실패에 반응. checkpoint를 관리하며, task에 장애가 발생했을시 복구를 시도한다. high-availability를 제공하기 위해 여러개의 JobManager를 가질 수 있다. 하나는 항상 leader가 되며 나머지 JobManager는 standby 상태가 된다(Zookeeper나 Kubernetes에 의해 서비스 가능). 3가지 component로 구성.
> * ResourceManager: standalone cluster, Kubernetes 등의 환경에서 리소스(task slot : 태스크 실행에 필요한 자원)를 관리.   
> * Dispatcher: flink application 실행을 위한 REST interface 제공.
> * JobMaster: single JobGraph 실행을 관리. flink cluster에서 여러 job이 동시에 돌아갈 때는 각각의 jobmaster를 가지고 있다.

* TaskManagers : 일반적으로 Flink 클러스터는 여러 태스크 매니저를 갖고 있음. 태스크매니저는 여러 태스크 슬롯을 가질 수 있음. 태스크 슬롯은 concurrent하게 실행가능. 하나의 task slot안에서 여러 개의 operator가 실행이 가능하다.

## Time
* flink는 data가 Streaming으로 들어왔을 때 시간에 대한 개념을 제공.
* Event Time: record가 device에게 생성된 시간 의미.
* Ingestion Time: record가 flink에 들어온 시간. 들어온 각각의 record에 대해 TimeStamp를 실시간으로 할당.
* Processing Time: record가 실제 flink에서 처리된 시간. 
* Time 개념들이 존재하는 이유: 만약 processing time을 기준으로 작업을 처리한다면, record가 네트워크나 source 문제로 지연도착을 했다면 window에서 data가 잘 못 처리 될 수 있다. 그래서 event time 기반의 데이터 처리를 위해 watermarks 개념이 이용. 

## Watermarks
* 일종의 timestamp로 window에서 watermark를 세팅할 수 있으며, window에 할당된 시간이 지나도 watermark 시간 안쪽우로 들어오면 해당 이벤트를 lateness로 간주.
* lateness인 데이터는 사용자 설정에 따라 버릴 수도 있고, 이전 window 처리에 포함할 수도 있다.

## CheckPoint
* Flink는 Exactly-Once를 보장하기 위해 내부적으로 CheckPoint 사용.
* 서비스 도중 장애가 발생했을 때, 장애 상황에 생긴 데이터도 완벽히 처리하는 것이 중요.
* stream 중간중간에 checkpoint barrier를 끼어넣음.
* 하나의 스트림으로 데이터가 들어오지만 빠른 데이터 처리를 위해 여러 파티션에서 데이터를 처리할 수 있다. 이 때 파티션마다 데이터가 빨리 처리되는 경우도 있고, 늦게 처리되는 경우가 발생.
* flink는 모든 파티션에 있는 데이터들이 barrier에 도달히여 정렬이 될 때까지 대기를 하게 됨. 정렬이 끝나면 새로운 체크포인트가 시작되고, 체크포인트 상태를 외부 저장소에 저장 후 다시 데이터 처리가 시작.
* 만약 장애가 발생했을 경우, 외부 저장소에 있던 체크포인트를 가져와서 체크포인트 시점의 상태를 복구. 그 시점의 스트림 offset부터 재처리.
* 그래서 데이터가 소비되더라도 데이터가 사라지지 않는 kafka 같은 브로커를 연결해서 사용.
* 데이터가 복구되면서 일부 레코드는 여러 번 sink가 되는데, 여러번 sink(저장)해도 동일한 내용을 저장할 수 있는(멱등성) 저장 방식으로 구성되어야 한다.


## To Do
* file sink 시 small file issue
