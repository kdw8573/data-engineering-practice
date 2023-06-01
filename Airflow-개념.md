- [Airflow 개념](#airflow-개념)
  - [ETL이란?](#etl이란)
  - [Apache Airflow](#apache-airflow)
  - [Airflow 등장 배경](#airflow-등장-배경)
  - [Core components](#core-components)
  - [DAG](#dag)
  - [Operator](#operator)
  - [구체적 스케쥴 예시](#구체적-스케쥴-예시)
  - [Logical(execution) date \& Schedule](#logicalexecution-date--schedule)
  - [Catch up](#catch-up)
  - [Default argument](#default-argument)
  - [Xcom](#xcom)
  - [Avoid top-level code in DAG file](#avoid-top-level-code-in-dag-file)
  - [Depends\_on\_past / Wait\_for\_downstream](#depends_on_past--wait_for_downstream)
  - [Sensor와 Trigger 차이](#sensor와-trigger-차이)
  - [Airflow Test](#airflow-test)

## Airflow 개념

### ETL이란?
* ETL(Extract, Transform, Load). 큰 조직에서 데이터를 정제하는데 쓰는 방법.
* 데이터의 소스가 한군데가 아니라 여러 곳에 퍼져있는 정제되지 않은 데이터를 추출(Extract)하고 용도에 맞게 변형(Transform)하고 그렇게 정제된 데이터를 한곳에 로드(Load)하는 프로세스를 의미.

### Apache Airflow
* ETL 워크플로우를 편리하게 해주는 도구.
* 프로그래밍 방식으로 워크플로우를 작성, 예약 및 모니터링하는 오픈 소스 플랫폼.
* ETL 처리 과정을 여러 개의 task로 나누고, 여러 개의 태스크를 하나의 파이프라인으로 만들어 스케쥴링 기능을 이용해 자동화까지 가능하게 만든 워크플로우 관리 도구.

### Airflow 등장 배경
* Airflow 없이도 일련의 task 처리 과정을 CRON과 쉘 스크립트를 통해 순차적으로 실행 및 스케쥴링이 가능하다.
* 그러나 잘 돌아가고 있는지 모니터링이 힘들었고, 그로 인해 상위 태스크가 잘 돌아가서 하위 태스크가 실행이 된 것인지 의존성 관리 문제도 있었다.
* 또한 에러에 대한 디버깅 및 재처리 작업의 어려움이 있었다. 
* 그로 인해 파이썬으로 쉽게 프로그래밍이 가능하며, 웹 UI를 통해 엔지니어가 쉽게 태스크를 관리(모니터링, 디버깅)할 수 있으며 여러대의 노드가 있는 분산된 환경에서도 적용할 수 있는 airflow가 등장하였다.
* Airbnb에서 2014년 10월 시작된 오픈소스 프로젝트 -> 오픈소스로 2015년 6월 발표 -> 2016년 3월 아파치 재단에서 인큐베이터 프로그램으로 뽑아 관리 시작 -> 2019년 1월 Airflow를 탑티어 프로젝트로 발표 

### Core components

* Scheduler : Workflow를 스케줄링하는 스케줄러 데몬이다. 모든 DAG와 태스크를 모니터링하고 관리하며, 주기적으로 실행해야 할 태스크를 찾고 해당 태스크를 실행 가능한 상태로 변경. 
 실행할 작업을 executor에게 제출. Airflow에서 가장 핵심이 되는 컴포넌트
* DAG Script : 개발자가 작성한 Python 워크플로우 스크립트
* Web Server : Airflow의 웹 인터페이스를 제공하는 웹 서버. Flask와 Gunicorn을 이용하여 인터페이스를 제공
* MetaStore(MetaDB) : 어떤 DAG가 존재하고 어떤 태스크로 구성되는지, 어떤 태스크가 실행 중이고 실행 가능한 상태인지 등의 메타데이터가 저장되는 데이터베이스. 주로 Postgresql을 추천하지만, SQL Alchemy와 호환 가능한 MySQL이나 SQLite도 이용이 가능하다.
* Executor : 어떤 환경에서 Task가 실행될지에 대한 타입 정의. 태스크 인스턴스를 실행하는 주체로, 다양한 타입이 존재한다.
> * Sequential Executor : Worker 가 한번에 하나의 Task만 수행할수 있어 제약적.
> * Local Executor : Worker 가 Scheduler 와 같은 서버에서 실행. 장점은 구성이 간단하여 베타 혹은 테스트 환경에 많이 사용. 단점은 단일 장비 환경에서 작동하기 때문에 Single point of Failure(SPOF) 발생가능. 
> * Celery Executor : 메시지 브로커(broker)가 필요하며, 메시지 브로커로는 RabbitMQ나 Redis를 사용할 수 있다. 스케줄러는 Task를 메시지 브로커에 전달하고, Celery Worker가 Task를 가져가서 실행하는 방식. 장점은 worker를 scale out 할 수 있다. 단점은 메시지 브로커가 생겨 관리 포인트가 늘어난다.
> * Kubernetes Executor : 컨테이너 환경에서 Worker 가 실행되도록 구성. Task를 스케줄러가 메시지 브로커에 전달하는게 아니라 Kubernetes API를 사용하여 Airflow 워커를 pod 형태로 실행. 태스크가 실행될 경우에만 워커를 생성하고 태스크가 완료되면 자원을 반납하기 때문에 Kubernetes의 자원을 효율적으로 사용있다는 장점이 있다. 그러나 상대적으로 구성이 까다롭다. 워커 POD는 휘발성이기 때문에 POD가 종료되면 로그가 유실. 그래서 외부 저장소인 Amazon S3나 Elastic Search 등으로 로그를 저장해야 된다. 
* Worker : 실제 Task를 처리하는 컴포넌트.
* Kerberos : 인증 처리를 위한 프로세스로 필수 사항은 아니다.

### DAG
* Directed Acyclic Graphs. 방향을 가진 비순환 그래프
* airflow에서는 순환하지 않고 시작에서 끝으로 진행되는 워크플로우 구조가 된다. 
* Task의 연결관계를 DAG로 관리하고, 웹 인터페이스를 통해서도 DAG 구조를 시각적으로 확인할 수 있다.

### Operator
* Operator에 특정한 인풋과 조건을 넣어주면 task가 된다.
* 즉, Task의 Wrapper 역할
* Action Operator: 기능이나 명령을 실행하는 Operator. ex) BashOperator, PythonOperator...
* Transfer Operator: 데이터를 전송하는 Operator.
* Sensor Operator: 특정 조건을 Sensing하여 실행되는 Operator. 다른 Operator들과는 달리 조건이 만족할 때까지 기다렸다가, 조건이 충족되면 다음 Task를 실행하도록 함.

### 구체적 스케쥴 예시
1. 새로운 파이프라인을 생성하여 dag.py를 dags 디렉토리에 추가하였다
2. Web server에서는 Dag를 웹 UI에 노출시킴. 
3. Scheduler는 Metastore에 DagRun Object(DAG의 인스턴스)를 생성한다.
4. Dag가 트리거되면, DagRun의 status가 Running으로 바뀌고, 진행해야할 Task가 TaskInstance로 Metastore에 생성된다.
5. TaskInstance가 생성된 이후, TaskInstance는 Scheduler에 의해서 Executor로 전달된다.
6. Executor가 TaskInstance를 실행한 이후에 Metastore의 TaskInstance 상태를 업데이트한다.
7. Scheduler는 지속적으로 Metastore를 확인하여 TaskInstance가 모두 종료되었는지 확인하고, 종료되었다면 DagRun의 status를 Completed로 업데이트
8. Webserver는 DagRun의 상태를 보고 웹 UI에서 업데이트

### Logical(execution) date & Schedule
* 전통적인 배치 데이터 처리 방식은 데이터의 기간이 있고, 그 기간이 지난 후 해당 기간에 발생한 데이터를 처리를 의미. 하루에 한번 실행되는 작업이라면 해당 되는 하루가 끝나고 날짜가 바뀔 때 작업이 실행 된다.
* 그로 인해 airflow에서 Logical date는 Task가 실제로 실행되는 시간이 아니라 해당 Task가 실행되는 Time Window의 시작 지점을 의미.
* 기존에는 명칭을 execution_date을 썼지만, 오해의 소지가 많았기 때문에 logical_date로 명칭을 바꾸었다. 작업이 실행되는 데이터 시작 날짜는 data_interval_start, 끝나는 날짜는 data_interval_end로 나누어 표현할 수 있게 되었다.
* 작업이 실행되는 스케쥴 간격은 Dag 속성 중에 schedule 인수를 통해 정의할 수 있다. preset을 제공하고 있다.
* 좀 더 복잡한 스케줄 간격을 지원하기 위해 MacOS/Linux 같은 Unix 기반 OS 스케줄러인 cron 구문을 지원한다.

    | preset  | meaning                                                          | cron      |
    |---------|------------------------------------------------------------------|-----------|
    | None	  | Don’t schedule, use for exclusively “externally triggered” DAGs	 |           |
    | @once	  | Schedule once and only once	                                     |           |
    | @hourly |	Run once an hour at the beginning of the hour	                 | 0 * * * * |
    | @daily  |	Run once a day at midnight	                                     | 0 0 * * * |
    | @weekly |	Run once a week at midnight on Sunday morning	                 | 0 0 * * 0 |
    | @monthly|	Run once a month at midnight of the first day of the month	     | 0 0 1 * * |
    | @yearly |	Run once a year at midnight of January 1	                     | 0 0 1 1 * |
* Airflow에서는 Jinja2 template를 내장하고 있어 이를 활용할 수 있다. macro를 통해 variable에 접근 가능하다.
* https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html 를 통해 확인가능. {{ ds }}는 logical date에 접근.

### Catch up
* Dag 속성 안에 Catch up이라는 인수를 줄 수 있다. 기본적으로 false가 되어있고, True로 주게 되면 Catch up이 활성화.
* 과거에 start_date를 설정하면 airflow는 과거의 Task를 차례대로 실행하는 Backfill을 실행

### Default argument
* Dag의 모든 task에 공통적으로 적용되는 arguments를 정의할 수 있다.
* Dag 속성의 default_args라는 인수에 dictionary 타입으로 줄 수 있다.

### Xcom
* Dag 내의 task 사이에서 데이터를 전달하기 위해서 사용.
* Variables와 마찬가지로 key-value의 형식으로 사용되지만, Variables과는 달리 Xcom은 DAG내에서만 공유할 수 있는 변수.
* DataFrame이나 많은 양의 데이터를 전달하는 것은 지원하지 않으며, 소량의 데이터만 전달하는 것을 권장.
* Xcom을 사용하기 위해 task_instance의 `xcom_push`나 `xcom_pull` method가 사용.

### Avoid top-level code in DAG file
* top-level code는 프로그램이 시작할 때 user가 지정한 파이썬 모듈 및 코드를 의미한다. 
* Airflow scheduler는 정해진 시간마다 dags 폴더에 있는 dag들을 parsing한다. 
* 그래서 시간에 따라 DAG의 스케쥴링이나 dependencies를 변경하거나 다음 스케쥴된 DAG에 영향을 줄 수 있다.(dynamic scheduling of the DAG)
* 그래서 계속 dag script를 읽게 되는데, 이 때 top-level에 적혀있는 코드가 계속 불리게 되며 이것은 DAG loading time에 많은 영향을 준다. 
* 가급적이면 Python 함수안에 local import로 만드는 것이 좋다. 대신 동일한 모듈이 여러번 사용되면 local import를 여러번 적어야 되는 수고가 생긴다.

### Depends_on_past / Wait_for_downstream
* DAG가 실행될 때 이전 DAG 작업을 이용하여 오늘 DAG가 실행될 경우, 이전 DAG에 대해 의존성이 생긴다.
* 그러나 airflow에 별 다른 설정이 없다면 이전 DAG가 실패하더라도 오늘 DAG는 정상적으로 실행이 된다.
* operator에 인수로 depends_on_past = True를 설정하면 이전 날짜의 task중에 어느 하나가 실패한다면, 다음 DAGrun에서는 실패 task전까지는 실행된 뒤, 이전에 실패한 task부터 대기한다. 이전 task가 성공한다면 다시 현재 DAGrun도 실행된다.  
* wait_for_downstream = True라면 이전 날짜의 task중에 어느 하나가 실패한다면, 다음 DAGrun에서는 running 상태이지만 task가 처음부터 실행이 되지 않는다. 실패한 task가 성공해야 현재 DAGrun이 실행된다.

### Sensor와 Trigger 차이
* 둘 다 operator의 종류. Sensor는 주어진 조건을 만족하는지 외부 이벤트를 주기적으로 체크할 때 사용한다.
* Sensor는 특정 조건을 만족할 때까지 기다리고, 조건을 만족하면 다음 task를 진행한다.
* Sensor의 부모 클래스인 BaseSensorOperator는 여러 옵션들이 존재한다.
> * poke_interval: 조건 확인의 재시도 주기. default 값은 60초.
> * timeout: 조건을 확인하는 최대 시간. 만약 timeout까지 조건을 만족하지 못하면 실패로 판정. default 값은 일주일.
> * mode: 어떻게 sensor operator가 작동하는지 타입을 정의하며 "poke"와 "reschedule" 2가지 타입이 존재. poke는 특정 조건을 만족할 때까지 worker를 점유. reschedule은 조건을 확인할 때만 worker를 점유한다. 그래서 만약 sensor가 long runtime을 가질 것 같다면, 자원 관리를 위해 reschedule 타입으로 정의를 하는 것이 좋다.
* Sensor중에 ExternalTaskSensor는 다른 DAG가 종료될 때까지 대기를 하며, DAG 사이에 종속성을 만들 수 있다.
* Trigger는 scheduling을 통하지 않고 DAG를 실행하는 것을 의미. 
* TriggerDagRunOperator는 외부 DAG를 호출 시키며, 외부 DAG가 성공할 때까지 기다릴 수도 있다.

### Airflow Test
* 단일 task에 대해 테스트를 진행하면 task 사이에 dependency를 고려하지 않고 실행시킨다. 그리고 상태값에 대해 DB에 반영하지 않는다.
* DAG 단위로 테스트 할 경우, task 사이의 dependency는 고려하지만 상태값은 DB에 반영하지 않는다.
* test시에 dry-run 옵션을 줄 수 있는데, task의 Template fields(ex. ds, ti, conn) 값만 렌더링이 가능한지 검사한다. 
