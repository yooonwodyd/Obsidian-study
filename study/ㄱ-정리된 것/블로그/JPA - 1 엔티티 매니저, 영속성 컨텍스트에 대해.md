

# 1.Entity Manager

## Entity Manger란 무엇인가?

> **The EntityManager is an API that manages the lifecycle of entity instances. An EntityManager object manages a set of entities that are defined by a persistence unit. Each EntityManager instance is associated with a persistence context.**

IBM에서 정의한 엔티티 매니저는,  엔티티의 인스턴스의 생명주기를 관리하는 API로, 
각 EntityManager 인스턴스는 영속성 컨텍스트와 연결되어 있는 것을 강조한다.

그렇다면, 엔티티 매니저는 어떻게 생성되고, 어떻게 데이터 베이스와 연결되며, 어떻게 쓰레드에서 동작할까?

## Entity Manager Factory의 역할
Entity Manager Factory는 이름에서 알 수 있듯이 Entity Manager를 생성한다.

뿐만 아니라 Entity Manager Factory는 persistence.xml이나 application.properties에 정의된 설정을 통해 데이터베이스를 연결한다.

데이터 베이스 연결시 JPA는 자체적인 커넥션 풀을 제공하지 않기 때문에, Springboot의 기본 설정의 경우 HikariCp를 통해 커넥션 풀을 관리한다.

이밖에 객체 매핑 전략을 관리하고, ddl-auto와 같은 설정 등을 관리한다.

**책 <자바 ORM 표준 JPA 프로그래밍>에 의하면, Entity Manager Factory는 생성하는데 매우 비용이 크기 때문에 하나만 생성해서 어플리케이션 전체에서 공유한다.**

## Entity Manager
Entity Manager Factory를 통해 Entity Manager를 생성했다면,
데이터베이스에 등록/수정/삭제/조회를 할 수 있다. 

또한 Entity Manager는 **Transaction** 내에서 유효하기 때문에, 트랙젝션이 끝나면 폐기되고, 필요할 때 다시 생성된다. 이때 Entity Manager는 Entity Manger Factory와 달리 **Thread Safe** 하지 않다.

왜 그런가 생각해보면, Connection을 받아서 소켓의 송신 버퍼에 값을 적재하려고 할 때, Thread safe 하지 않다면 적재되는 값이 섞일 수 있기 때문이라 추측해 볼 수 있다.(*엔티티 매니저는 데이터베이스 커넥션과 밀접한 관계가 있으므로 스레드 간에 공유,재사용 되면 안된다 - JPA 프로그래밍*)

Entity Manager의 주요한 특징 중 하나는 영속성 컨텍스트를 통해 다양한 이점을 누릴 수 있다는 것인데,
이를 이해하기 위해선 영속성 컨텍스트가 무엇인지 알아야 할 것이다.


## 영속성 컨텍스트(persistence context)

> **A persistence context is a set of entity instances in which for any persistent entity identity there is a unique entity instance. Within the persistence context, the entity instances and their lifecycle are managed. The `EntityManager` API is used to create and remove persistent entity instances, to find entities by their primary key, and to query over entities.**

오라클 공식문서의 표현을 빌리면, 영속성 컨텍스트는 고유한 엔티티 인스턴스를 저장하는 환경이라 할 수 있다.
**책 <자바 ORM 표준 JPA 프로그래밍>** 에서는 영속성 컨텍스트를 논리적인 개념에 가깝고 눈에 보이지 않는다고 하는데, 이는 메모리 영역에서 엔티티를 저장해둔 영역을 통틀어 말하기 때문으로 보인다.

## 그렇다면 해당 엔티티가 고유한 인스턴스인지 어떻게 판단하는가?
영속성 컨텍스트는 엔티티를 식별자 값으로 구분한다. 이때 식별자 값은 테이블의 기본 키와 매핑된 값이다.
이 정보를 **1차 캐시**라고 하는데, 이를 통해 다음과 같은 이점을 누릴 수 있다.

1. **쓰기지연** : 트랜잭션을 커밋하기 전까지, 1차 캐시에 엔티티의 변경사항들을 저장해둔다. 이후, 커밋이 완료되면 이에 맞게 내부 쿼리 저장소에 저장해둔 SQL을 데이터베이스에 보낸다.
2. **조회 속도 향상**: 1차 캐시에 이미 존재하는 값이라면, 굳이 데이터베이스에서 조회 쿼리를 통해 조회하지 않아도 엔티티의 정보를 가져올 수 있다.
3. **변경 감지**: 최초 데이터베이스에서 정보를 불러왔을 때의 엔티티 상태를 별도로 저장해둔다. 이를 스냅샷이라고 하는데, 플러시 시점에 스냅샷과 엔티티를 비교해서 변경된 엔티티가 있으면 수정 쿼리를 생성한다.
4. 지연 로딩

위에서 플러시를 간단하게 설명했는데, 플러시는 영속성 컨텍스트의 변경 내용을 데이터 베이스에 반영한다.
이때 변경 감지와 함께, 쓰기 지연 SQL 저장소의 쿼리를 데이터 베이스에 전송하는 두가지 일을 한다.