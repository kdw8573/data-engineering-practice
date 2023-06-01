- [Unit Test 개념](#unit-test-개념)
  - [Unit Test(단위 테스트)란](#unit-test단위-테스트란)
  - [Unit Test 필요성](#unit-test-필요성)
  - [FIRST 5가지 규칙](#first-5가지-규칙)
  - [Give-When-Then 패턴](#give-when-then-패턴)
  - [Code Coverage](#code-coverage)
  - [Mocking](#mocking)
  - [unittest](#unittest)

# Unit Test 개념

## Unit Test(단위 테스트)란
* 코드를 작성할 때 특정 모듈(메소드)이 의도한 대로 작동하는지 검사하는 가장 작은 단위의 테스트.
* 즉, 작성한 모든 메소드에 대해 테스트케이스를 작성.
* 반대로 Integration Test(통합 테스트)는 모듈 간의 연계가 올바르게 되어 동작하는지 검증하는 테스트.
* 통합 테스트는 DB 등 다른 컴포넌트와 실제 연결을 해야 되고, 시스템을 구성하는 컴포넌트들이 많으면 연결에 시간이 오래 걸림.
* 실무에서는 테스트 코드는 대부분 독립적으로 실행되는 단위 테스트가 일반적.

## Unit Test 필요성
* 단일 메소드에 대해 테스트를 진행하기 때문에 새로운 기능 추가 시에 수시로 빠르게 테스트 가능.
* 코드 리팩토링 시, 빠르게 문제 여부를 판단 가능.
* 따라서 테스팅에 대한 시간과 비용을 절감 가능.

## FIRST 5가지 규칙
* 좋은 테스트 코드는 FIRST라는 5가지 규칙을 따라야 한다.
1. Fast: 테스트는 빠르게 동작하여 자주 돌릴 수 있어야 한다.
2. Independent: 각각의 테스트는 독립적이며 서로 의존해서는 안된다.
3. Repeatable: 어느 환경에서도 반복 가능해야 한다.
4. Self-Validating: 테스트는 성공 또는 실패로 bool 값으로 결과를 내어 자체적으로 검증되어야 한다.
5. Timely: 테스트는 실제 코드를 구현하기 직전에 구현해야 한다.

## Give-When-Then 패턴
* Unit Test를 작성할 때 쓰이는 패턴.
* given(준비): 어떠한 데이터가 준비되었을 때
* when(실행): 어떠한 함수를 실행하면
* then(검증): 어떠한 결과가 나와한다.

## Code Coverage
* 소프트웨어 테스트를 진행했을 때 코드 자체가 얼마나 실행되었는지를 나타내는 수치.
* 테스트 커버리지 수치를 통해 코드에서 발생할 수 있는 모든 시나리오를 검사하는 것이 목적.
* 코드 커버리지 기준은 크게 3가지가 존재.
* Statement Coverage: 코드 한 줄이 한 번이상 실행되는지 확인.
* Decision(Branch) Coverage: 조건문에 대해서 테스트 케이스들이 true와 false를 둘 다 검사하는지 확인.  
* Condition Coverage: 테스트 케이스를 통해 조건문 내부에 있는 조건들이 각각 true, false를 검사하는지 확인. 

## Mocking
* Unit Test code 작성 시에 외부 DB나 API에 의존하는 코드를 테스트해야 될 때가 생김.
* 제약이 많은 테스트 환경에서는 연동이 어려운 경우가 있고, 실행 속도도 느림.
* 애플리케이션 객체의 동작을 모방하는 시뮬레이션된 객체인 Mock 객체를 이용해 외부에 의존하는 부분을 mock 객체로 대체하는 기법 사용(Mocking).
* mock객체에 `return_value`옵션을 통해 호출되었을 때 리턴값을 정의할 수 있음. ex) mock = Mock(return_value = 'hi')
* `side_effect`옵션을 통해 예외 상황이나, 주어진 인자 값에 따라 리턴하는 값을 정의할 수 있음. ex) mock = Mock(side_effect=lambda x: x * 10)
* Mock 객체는 파이썬의 매직 메소드를 자동으로 정의하지 않는다. 그러나 MagicMock 클래스를 사용하면 매직 메소드를 알아서 정의해주므로, 따로 메소드에 대해 mocking할 필요 없이 return_value만 설정하면 된다. 
* patch 데코레이터를 통해 특정 모듈의 함수나 클래스를 MagicMock 객체로 대체할 수 있다.

## unittest
* Unit Test 작성을 위한 파이썬 내장 모듈.
* unittest.TestCase를 상속받아 Custom Test Class를 정의.
* `test_` 시작하는 함수를 정의하면 해당 함수를 테스트.
* setUp(): 각 테스트 메소드를 호출하기 이전에 호출되는 메소드. 테스트 케이스가 실행될 때 마다 사용.
* tearDown(): 각 테스트가 끝난 이후에 호출되는 메소드. setUp 메소드가 성공했을 경우에만 호출.
* setUpClass(): 해당 테스트 클래스가 시작되기 이전 단 한번 호출되는 메소드.
* tearDownClass(): 해당 테스트 클래스가 종료된 이후 단 한번 호출되는 메소드.