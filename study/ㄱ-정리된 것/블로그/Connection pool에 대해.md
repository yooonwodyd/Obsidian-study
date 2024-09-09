
# DB Connection Pool이란

> 데이터베이스 커넥션 풀(Database Connection Pool)은 데이터베이스와의 연결(Connection)을 미리 여러 개 생성해 두고, 필요할 때마다 이를 재사용하는 기술입니다. 왜냐하면 매번 데이터베이스 연결을 새로 생성하는 것은 시간과 자원을 많이 소모하기 때문입니다.(https://f-lab.kr/insight/understanding-database-connection-pool 참조)

이때의 커넥션 풀은 LISTEN 상태나 SYN-RECEIVED 상태의 소켓이 아니라,
3-way-handshake를 통해 송신 버퍼와 수신 버퍼를 메모리에 탑재한 Established한 소켓을 말한다. 

## 번외. TCP/IP 연결의 3-way handshake

### TCP는 Connection-oriented 통신이다.
Connection-oriented 통신이라는 건, 통신을 하기 전에 연결을 위한 설정이 존재한다는 뜻이다. TCP 프로토콜을 예시로 들자면 **3-way handshake** 이다.

- **1단계 (SYN)**: 클라이언트가 서버에게 연결 요청 메시지(SYN)를 보낸다.
- **2단계 (SYN-ACK)**: 서버는 연결 요청을 수락하고, 이를 확인하는 메시지(SYN-ACK)를 클라이언트에게 보낸다.
- **3단계 (ACK)**: 클라이언트는 서버의 수락을 확인한 후, ACK 메시지를 보내 연결이 성립된다.

이렇게 3-way-handshake를 마치면 클라이언트와 서버는 서로의 IP 주소와 서로의 포트 번호를 알게 된다.  따라서 소켓 연결이 가능해진다.


# HikariCP와 쓰레드

> Fast, simple, reliable. HikariCP is a "zero-overhead" production ready JDBC connection pool. At roughly 130Kb, the library is very light. Read about [how we do it here](https://github.com/brettwooldridge/HikariCP/wiki/Down-the-Rabbit-Hole). (히카리 공식 문서)

HikariCP는 JDBC Connetion Pool이다. 자바의 JDBC API에 맞춰서 설계되었다.
가장 범용적으로, 많이 사용되는 JDBC  Connection Pool의 구현체이다.

해당 HikariCP에서, 실제로 pool을 가져오는 과정을 코드로 보자.


![[스크린샷 2024-09-08 오후 4.14.10.png]]
다음 코드에서 핵심만 보면, while문을 돌며 타임아웃이 나기 전까지 poolEntry를 요청하고, 실제로 connection.Bag에서 poolEntry가 왔다면 유효한지 확인한 후 커넥션 프록시를 생성해서 반환한다. 때문에 커넥션을 닫아도 실제로 커넥션이 닫히는 것이 아닌 커넥션 풀로 커넥션이 반환된다.

poolEntry를 받아오는 this.connectionBag.borrow의 코드에선 쓰레드에서의 최적화나 타임아웃 등 핵심적인 설정이 들어가있다. JVM에 대한 이해가 추가로 필요해보여서 완전히 이해는 못했지만, 우아한 테크 블로그(https://techblog.woowahan.com/2664/)에 기술된 플로우 차트를 보면 다음과 같은 순서로 작동한다.

1. 현재 쓰레드에 이전에 사용한 커넥션 정보가 있는지 확인한다.
2. 이전에 사용한 커넥션 리스트 중에서, 현재 사용가능한 커넥션이 있다면 해당 커넥션을 리턴한다.
3. 현재 쓰레드가 아닌, 전체 쓰레드 풀에서 사용 가능한 커넥션이 있는지 확인한다.
4. 만약 현재 사용가능한 커넥션이 없다면, handOffQueue에서 사용가능한 커넥션이 있을 때 까지 기다린다.
5. 만약 Timeout이 발생하면 null을 리턴한다.

여기서 null이 리턴되면, 에러를 발생시키고 락을 반환한다.(해당 코드는 getConnection에 존재.)

## Hikari CP과 데이터베이스

HikariCP는 앞서 말했듯이 JDBC Connetion Pool이다.
다시 말해 **어플리케이션 내부의 커넥션 풀을 관리**한다.

**이는 데이터 베이스의 커넥션 풀과는 별도라는 것을 의미한다.**
만약 AWS의 RDS에 데이터베이스를 두었다고 하면, **HikariCP의 커넥션 풀 크기**는 **AWS RDS에서 허용하는 최대 커넥션 수를 초과할 수 없을 것이다.

추가로, 어플리케이션 서버를 자주 재시동한다면 AWS의 커넥션 풀의 개수가 부족하여 앱에서 오류가 발생할 수 있다. 

때문에 어플리케이션의 적절한 커넥션의 개수를 고민함과 동시에, 현재 배포한 데이터베이스의 성능 또한 고려할 줄 알아야 할 것이다.




