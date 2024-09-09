### Entity Manger란 무엇인가?

> The EntityManager is an API that manages the lifecycle of entity instances. An EntityManager object manages a set of entities that are defined by a persistence unit. Each EntityManager instance is associated with a persistence context.

IBM에서 정의한 엔티티 매니저는,  엔티티의 인스턴스의 생명주기를 관리하는 API로, 
각 EntityManager 인스턴스는 영속성 컨텍스트와 연결되어 있는 것을 강조한다.

그렇다면, 엔티티 매니저는 어떻게 생성되고, 어떻게 데이터 베이스와 연결되며, 어떻게 쓰레드에서 동작할 까?


# 1.Entity Manager Factory

## Entity Manager Factory란
엔티티 매니저를 생성하기 위해선 Entity Manager Factory가 필요하다.

책 <자바 ORM 표준 JPA 프로그래밍>에 의하면, 이런 Entity Manager Factory는 생성하는데 매우 비용이 크기 때문에 하나만 생성해서 어플리케이션 전체에서 공유한
다.

Entity Manager Factory는 persistence.xml이나 application.properties에 정의된 설정을 통해 데이터베이스를 연결한다.  JPA는 자체적인 커넥션 풀을 제공하지 않기 때문에, Springboot의 기본 설정의 경우 HikariCp를 통해 커넥션 풀을 관리한다.
또한 객체 매핑 전략을 관리하고, ddl-auto와 같은 설정을 관리한다.

## Entity Manager
Entity Manager Factory를 통해 Entity Manager를 생성했다면, 데이터베이스에 등록/수정/삭제/조회를 할 수 있다. 또한 **Transaction** 내에서 유효하기 때문에, 트랙젝션이 끝나면 폐기되고, 필요할 때 다시 생성된다.



### 영속성 컨텍스트
1차 캐시의 키는 식별자 값이다.(@Id)





