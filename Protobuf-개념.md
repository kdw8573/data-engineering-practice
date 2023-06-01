- [Protobuf 개념](#protobuf-개념)
  - [Protobuf](#protobuf)
  - [Protobuf 원리](#protobuf-원리)
  - [protobuf 특징](#protobuf-특징)
  - [protobuf 사용법](#protobuf-사용법)
  - [proto3 문법](#proto3-문법)

# Protobuf 개념

## Protobuf
* Protocol Buffers(Protobuf)는 Google에서 개발하고 오픈소스로 공개한 데이터 직렬화 형식.
* 구조화된 데이터를 binary 형식으로 효율적으로 저장하여 네트워크에 빠르게 전송.
* C++, Java, Python, Ruby 등 다양한 프로그래밍 언어를 지원하고, 특정 언어 종속성이 없는 proto file로 작성된 데이터 타입을 다른 언어로 쉽게 포팅가능.

## Protobuf 원리
```
message Person {
    required string user_name   = 1;
    optional int64  number      = 2;
    repeated string interests   = 3;
}
```
* 위는 .proto 파일에 적힌 예시. 속성이름을 숫자로 대체한다.
* 데이터를 변환할 때 1byte는 데이터 태그 번호(5bit)와 type(3bit)를 나타낸다.
* 그 다음 1byte는 데이터 길이를 표시하며, 나머지 byte들은 데이터를 16진수로 인코딩한 값이다.

## protobuf 특징
* 통신이 빠르다. JSON, XML에 비교해서 데이터 크기가 작기 때문에 같은 시간에 더 많은 데이터를 보낼 수 있다.
* 사람이 읽기 불편하다. JSON은 텍스트 형식으로 key-value 구조를 가져 사람이 읽기 편하지만 프로토콜 버퍼가 쓴 데이터는 사람이 읽기 어렵다. proto 파일를 보지 않으면 의미 파악이 어려움.
* proto 문법을 배워야 데이터를 직렬화 할 수 있다.

## protobuf 사용법
* proto 파일에 원하는 데이터 포맷을 작성.
* protoc 컴파일러로 원하는 언어 형식으로 컴파일.
> * protoc -I=./ --python_out=./ ./address.proto
> * -I에는 이 protofile이 있는 소스 디렉토리, --python_out에는 생성된 파이썬 파일이 저장될 디렉토리, 마지막로 proto 파일 위치.
* 해당 언어로 생성된 데이터 클래스를 이용해 프로그램에 적용.

## proto3 문법
* protobuf에 버전2가 있고, 버전3가 있다. proto2가 default version이지만, protoc3가 최신버젼.
* `syntax = "proto3";` 명시를 통해 proto3 문법을 사용함을 나타냄.
* field type은 scalar 타입(int32, string, double..)이나 enum(열거형), 다른 메시지 타입을 가질 수 있다.
* field 번호는 고유의 번호를 할당하는데, 1부터 15까지는 1byte를 필요로 하고, 그 이후부터는 2byte 이상을 쓰므로 자주 쓰는 필드의 경우에는 1부터 15까지 번호를 할당토록 한다.
* field rule은 singular 또는 repeated이다. singular는 메시지에 그 필드가 한 번이상 나타나지 않고, repeated는 메시지에 그 필드가 여러번 등장할 경우 사용한다.
* field 타입을 모를경우에는 Any 타입을 이용하여 메시지에 포함할 수 있다.  
* `option java_multiple_files = True`로 할 경우 각 메시지의 대한 java 파일이 생성. 자바로 컴파일하지 않을 경우에는 해당 옵션은 무시.  
* wrapperType(StringValue, Int32Value)을 통해 default value와 null value를 구분할 수 있다.