- [Spark 개념](#spark-개념)
  - [spark 등장 배경](#spark-등장-배경)
  - [spark architecture](#spark-architecture)
  - [spark mode](#spark-mode)
  - [Cluster Manager](#cluster-manager)
  - [RDD(Resillient Distributed Data)](#rddresillient-distributed-data)
  - [DataFrame \& DataSet](#dataframe--dataset)
  - [Transformation \& Action](#transformation--action)
  - [실행 계획 최적화](#실행-계획-최적화)
  - [Partition](#partition)
  - [Shuffle](#shuffle)
  - [Job, Stage, Task](#job-stage-task)
  - [SparkSession, SparkContext](#sparksession-sparkcontext)

# Spark 개념

## spark 등장 배경
* Spark는 하둡의 MapReduce 형태의 클러스터 컴퓨팅 한계를 극복하고자 등장. 
* MapReduce는 Disk로부터 데이터를 읽은 후, Map을 통해 흩어져 있는 데이터를 Key-Value 형태로 연관성 있는 데이터끼리 묶은 후에, Reduce를 하여 중복된 데이터를 제거하고, 원하는 데이터로 가공하여 다시 Disk에 저장.
* 하지만 파일 기반의 Disk IO는 성능이 좋지 못했고, In-memory의 연산을 통해 처리 성능을 향상시키고자 Spark가 등장.
* Apache Spark는 오픈소스이며, 범용적인 분산 클러스터 컴퓨팅 프레임워크. 클러스터 환경에서 데이터를 병렬로 고속 처리.
* In-memory 연산을 통해 디스크 기반의 Hadoop에 비해 성능을 약 100배 정도 향상.

## spark architecture
* 스파크 어플리케이션과 클러스터 매니저로 구성. 사용자는 클러스터 매니저에게 스파크 애플리케이션을 제출. 클러스터 매니저는 제출받은 애플리케이션 실행에 필요한 자원을 할당하고, 스파크 애플리케이션은 할당받은 자원으로 작업을 처리.
* Spark Application
> * Driver 프로세스와 다수의 Executor 프로세스로 구성. 즉, Driver와 Executor를 합쳐서 Spark application.
>> * Driver는 사용자의 Main 함수, 사용자가 작성한 로직을 실행. 이 과정에서 실행 계획을 생성해 Executor들에게 Task를 할당.
>> * Cluster Manager와 통신하며 Spark Application을 관리
>> * Spark Application 정보의 유지 관리, 사용자 프로그램이나 입력에 대한 응답, 전반적인 Executor 프로세스의 작업과 관련된 분석, 배포, 스케줄링 역할을 수행.
> * Executor
>> * 다수의 worker 노드에서 실행되는 프로세스.
>> * Driver에서 요청한 작업을 분산 처리 하거나 cache에 데이터를 분산 저장한 값을 들고 있음.
* Cluster Manager
> * Driver를 통해 Spark가 Cluster Manager에 연결되면 Executor 실행에 필요한 리소스를 얻어올 수 있음.
> * 다양한 Cluster Manager를 지원(Standalone, Hadoop Yarn, Kubernetes)


## spark mode
* Spark local mode
> * local mode는 클러스터를 사용하지 않고, 로컬 단일 머신에서 모든 것을 실행하는 형태.
> * Cluster와 Cluster Manager가 없기 때문에, 단일 머신 환경을 구축하거나 간단한 테스트를 할 때 유용.
> * Local Client JVM에 Driver 1개와 Executor 1개를 생성하는 형태. Executor 내부에는 여러 개의 Core를 사용하여 태스크를 병렬로 실행.

* Cluster Manager는 2가지 배포 방식 존재. Spark Driver를 어디에서 실행시키느냐 따라 다름.
* deploy mode - Client mode
> * Spark 실행 시 Driver가 Cluster 외부에서 실행되는 모드.
> * Driver Program은 Client 프로세스에 존재. 따라서 Spark Application을 실행했던 콘솔을 닫아 버리거나 기타 다른 방법으로 Client 프로세스를 중지시키면, Spark Context도 함께 종료되면서 수행 중이던 모든 스파크 job이 중지.
> * 따로 지정하지 않으면 기본으로 선택되는 모드. 

* deploy mode - Cluster mode
> * Driver가 Cluster 내의 Worker 노드 중 하나에서 실행되는 모드.


## Cluster Manager
* standalone mode
> * Spark 자체의 Cluster Manager를 사용하는 모드.
> * Executor는 Worker 노드 하나당 한 개씩 동작.
* Yarn - client
> * Driver는 외부의 client에서 동작. Executor는 Yarn의 Node Manager 내부의 container에서 동작.
> * Cluster Manager는 Resource Manager가 되며, Node Manger에 Executor를 위한 Container를 할당.

* Yarn - cluster
> * Driver가 Cluster 내부의 Node Manager에서 동작.
> * 그 외에는 client와 동일
> * 1. Client가 Spark Application을 Resource Manager에게 제출.
> * 2. Resource Manager는 NodeManager 중 하나를 선정해서 Application master(Driver)를 실행할 컨테이너를 할당하라고 지시.
> * 3. NodeManager는 Application master(Driver)의 컨테이너를 시작.
> * 4. Application master(Driver)는 스파크 executor에 사용할 컨테이너들을 리소스 매니저에 추가로 요청.
> * 5. Resource Manager가 리소스 할당을 허락하면, Application master(Driver)는 Node Manger에게 컨테이너를 시작하라고 지시.
> * 6. NodeManager는 스파크 executor에서 사용할 컨테이너를 시작.
> * 7. driver와 executor는 직접 통신하면서 스파크 어플리케이션을 수행.


## RDD(Resillient Distributed Data)
* Resillient(회복력 있는, 변하지 않는) : 메모리 내부에서 데이터가 손실 시 유실된 부분을 재연산해 복구
* Distributed(분산된) : 스파크 클러스터를 통하여, 메모리에 분산되어 저장
* RDD는 불변의 특성을 가지므로(Read-Only), 특정 동작을 처리하기 위해서는 기존 RDD에서 변형을 가한 새로운 RDD가 생생될 수 밖에 없음.
* 이 때 사용되는 연산자는 Transformation, Action 연산자가 있음.
* 따라서 스파크 연산을 함에 따라 수많은 RDD가 생성. 이 때 생성되는 연산 순서가 RDD Lineage.
* RDD Lineage는 DAG 구조를 가지고 있고, 특정 RDD 정보가 메모리에서 유실될 경우 그래프를 복기하여 자동으로 복구 가능.
* 이러한 특성으로 Fault-tolerant 특성을 가짐.
* 저수준의 데이터 형태로 스키마가 없음.
* 별도의 내장된 최적화(Optimize) 엔진이 없음. 사용자는 테이블 조인이나 효율화 처리를 사용자가 직접 제어.


## DataFrame & DataSet
* DataFrame 
> * Spark 1.3부터 등장. 스키마를 가진 RDD. RDD의 resilient, distributed하다는 특징을 상속. RDD처럼 한번 정의하면 변경 불가능.
> * DataFrame은 테이블의 데이터를 로우와 칼럼으로 단순하게 표현. 
* DataSet
> * spark 1.6부터 등장. DataFrame의 확장된 버전. 
> * Java와 Scala에서만 사용 가능 (Python에서 사용불가)
> * 타입 safe한 특성. spark 2.0부터는 DataFrame과 DataSet이 DataSet으로 병합.
* Catalyst Optimizer를 통해 실행 시점에 코드 최적화를 하여 RDD로 코드가 생성된 후 실행.

## Transformation & Action
* RDD 변경방법을 spark에게 알려줄 때 쓰이는 명령이 Transformation.
* Transformation을 반복할수록 filter, map 등의 연산을 통해 어떻게 가공할지 실행 계획이 누적됨.
* Narrow Transformation: 데이터의 이동이 필요 없는(Shuffle이 발생하지 않는) Transformation. 각 입력 Partition이 하나의 출력 Partition에만 영향. ex) map, filter
* Wide Transformation: 하나의 입력 Partition이 여러 출력 Partion에 영향. ex) SQL의 group by 처럼 특정 키를 기준으로 데이터를 모은 후 집계하는 경우


* 마지막으로 최종 RDD에 대해 어떠한 행동을 취할지 나타내는 것이 Action.
* Action을 실행하는 순간 이제까지 적용했던 Transformation이 적용(Lazy Evaluation).
* Lazy Evaluation을 통해 연산을 바로 적용하지 않고, 실행 계획을 분석하여 최적화 한 뒤에 물리적으로 실행.  


## 실행 계획 최적화
1. DataFrame/DataSet/SQL을 사용해 코드를 작성.
2. 스파크가 코드를 Logical plan으로 변환
> * 카탈리스트 옵티마이저는 logical plan을 Predicate Pushdown, Projection Pruning을 이용해 최적화된 logical plan을 만듬.
> * Predicate Pushdown: 데이터 전체를 가져온 후에 필터링하는 것이 아니라 저장소에서 데이터를 필터링이 된 결과물만 가져와 네트워크 및 가공 비용을 줄임.
> * Projection Pruning: 필요한 칼럼만 뽑아오는 것
3. 논리적 실행 계획을 물리적 실행 계획(Physical plan)으로 변환하여 최적화
> * 논리적 실행 계획을 클러스터 환경에서 실행하는 방법을 정의(ex. Logical Plan 에서는 Join -> Physical Plan 에서는 Sort-merge Join)
> * 다양한 실행 전략을 생성하고 비용 비교 후 최적의 전략을 선택.
> * 구조적 API(DataFrame, Dataset, SQL)를 일련의 RDD와 트랜스포메이션으로 변환.
4. 클러스터에서 물리적 실행 계획(RDD 처리)을 실행.


## Partition
* 일반적으로 Spark는 단일 머신에서 처리하기 어려운 큰 사이즈의 데이터를 사용
* 사용자는 DataFrame 같은 추상화된 API를 통해 데이터를 하나처럼 다룸
* 그러나 Spark는 데이터를 분할해 Partition 단위로 처리
* Partition이 많고, Executor도 많을 경우 동시에 여러 Executor에서 처리가 될 수 있으므로 빠름. Executor 숫자가 적다면 Partition을 아무리 잘게 쪼개도 병렬 처리가 불가능.
* 특정 데이터를 기준으로 Group By 등을 수행하는 경우가 많아 데이터 처리가 늦어짐. 특정 칼럼을 기준으로 파티션을 만들면 Transformation을 수행할 때 데이터 이동이 적게 발생할 수 있다.

## Shuffle
* 특정 연산을 수행하기 위해 여러 Partition 내의 데이터들이 그룹화되어 다른 Partition 들로 이동하는 것. Wide Transformation.
* 어떤 데이터를 이동해야 할지 모르므로 전체 데이터에 대한 탐색이 필요할 수 있음.
* Shuffle로 인해 Memory / Disk / Network 등 많은 자원이 소모하므로 Shuffle 을 적게, 필요한 만큼만 수행하는 것이 중요


## Job, Stage, Task
* Job: 1회성으로 실행되는 작업. Spark Job은 Airflow 같은 스케쥴러에서 주기적으로 실행되어 할일을 마친 뒤 종료되는 Spark Application.
* Task: 실행계획은 RDD operation으로 바뀌고, RDD operation은 물리적 작업 단위인 Task로 변환. 
* Stage: Task의 묶음. Stage를 구분하는 기준은 Wide Transformation(Shuffle)
* Stage 내에서 Task는 병렬로 실행될 수 있으나, Stage는 Shuffle로 구분되고 Shuffle은 데이터 이동을 하므로 다음 Stage가 시작되기 위해서는 이전 Stage가 종료되어야함.


## SparkSession, SparkContext
* SparkApplication의 entry point로 스파크의 기능들과 구조들이 상호방식하는 방식을 제공
* SparkContext: spark2.0 이전에 SparkContext가 모든 Spark 애플리케이션의 진입점. RDD 생성시 필요.
* SparkSession: spark2.0 이후에 모든 Spark 기능의 진입점. SparkContext에서 사용할 수 있는 기능도 모두 사용 가능. DataSet/ DataFrame 생성시 이용.
