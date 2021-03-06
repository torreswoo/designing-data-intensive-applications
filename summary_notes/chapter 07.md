# 7. 트랜잭션

- 현실 세계에서 데이터 시스템은 여러 문제가 생길 수 있다
  - SW나 HW의 실패
  - 앱의 사망
  - 네트워크 단절
  - 쓰기 충돌
  - 부분적 갱신
  - 경쟁 조건(?)
- 수십년 동안 **트랜잭션**은 이런 문제를 단순화하는 매커니즘으로 채택돼왔다
  - 트랜잭션: 애플리케이션에서 몇 개의 읽기와 쓰기를 하나의 논리적 단위로 묶는 것
  - 프로그래밍 모델을 단순화하려는 목적에서 만든 것
  - 한 트랜잭션 내의 모든 읽기와 쓰기는 한 연산으로 시행된다
  - 전체가 성공하거나 실패하거나 둘 중 하나
  - 트랜잭션이 실패하면 애플리케이션에서 안전하게 재시도한다 
  - 부분적인 실패를 걱정할 필요가 없기 때문에 오류 처리가 훨씬 단순해짐
  - 잠재적인 오류 시나리오와 동시성 문제를 무시할 수 있다 -> 안전성 보장

- 트랜잭션이 항상 필요하지는 않기 때문에 정확한 이해와 함께 적시적소에 사용하는 게 중요하다

<br>

## 7.1. 애매모호한 트랜잭션의 개념

- 현대의 거의 모든 RDB는 트랜잭션을 지원한다
  - 2000년대 후반에 인기를 얻은 NoSQL의 다수는 트랜잭션을 완	전히 포기하거나 과거보다 훨씬 약한 보장을 하도록 재정의했다.
- 트랜잭션은 필수적인 요구사항도, 확장성의 안티테제도 아니다

### 7.1.1. ACID의 의미

- 트랜잭션이 제공하는 안전성 보장은 원자성, 일관성, 격리성. 지속성을 의미하는 약어인 ACID로 알려져있다
- ACID 표준을 따르지 않는 시스템을 BASE라고 하기도 하는데 별 현실적 의미는 없다

**원자성**

- *다중 스레드 프로그래밍*에서 한 스레드가 원자적 연산을 실행한다면, 다른 스레드에서는 절반만 완료된 연사을 관찰할 수 없다
  - 시스템은 연산을 실행하기 전이나 후의 상태에만 있을 수 있으며 그 중간 상태에 머물 수 없다
- 하지만 ACID의 원자성은 동시성과 관련이 없다
- ACID의 원자성은 클라이언트가 쓰기 작업 몇 개를 실행하려고 하는데 그중 일부만 처리한 후 경함이 생기면 무슨 일 쌩기는지 설명한다
- 하나의 원자적 트랜잭션에 묶인 여러 쓰기 작업이 결함 때문에 완료될 수 없다면
  - 어보트되고 DB는 이 트랜잭션에서 지금까지 실행한 쓰기를 무시하거나 취소한다

**일관성**

- 굉장히 여러 의미로, 최소 네 가지 의미로 쓰이고 있다.
  - 복제 일관성과 비동기식으로 복제되는 시스템에서 발생하는 최종적 일관성
  - 재균형화를 위해 사용하는 파티셔닝 방법인 일관성 해싱
  - CAP에서 선형성을 의미하는 일관성
  - ACID의 맥락에서 일관성은 데이터 베이스가 '좋은 상태'에 있어야 한다는 걸 의미한다
- ACID 일관성의 아이디어: 한상 진실이어야 하는 데이터에 관한 어떤 선언(**불변식**)이 있다
  - 예컨대 회계 시스템에서 모든 계좌에 걸친 대변과 차변이 항상 맞아야 한다
- 일관성의 아이디어는 애플리케이션의 불변식 개념에 의존하지, DB가 보장하는 것이 아니다

**격리성**

- 클라이언트들이 동일한 DB 레코드에 접근하면 동시성 문제에 맞닥뜨리게 된다
- 격리성: 동시에 실행되는 트랜잭션은 서로 격리된다

- 실제로는 여러 트랜잭션이 동시에 실행됐더라도 트랜잭션이 커밋됐을 때의 결과가 트랜잭션이 순차적으로 실행됐을 때의 결과와 동일하도록 보장한다.
- 하지만 성능손해 때문에 잘 안 쓰고 대신 스냅숏 격리를 쓰기도 한다

**지속성**

- 트랜잭션이 성공적으로 커밋됐다면 하드웨어 결함이 발생하거나 데이터베이스가 죽더라도 트랜잭션에서 기록한 모든 데이터는 손실되지 않는다는 보장이다.
- 단일 DB에서 지속성은, 일반적으로 데이터가 비휘발성 저장소에 기록됐다는 뜻
  - 보통 복구할 수 있는 WAL 등의 수단을 동반한다
- 복제 기능이 있는 DB에서 지속성은, 데이터가 성공적으로 다른 노드에 복사됐다는 것
- 완벽한 지속성은 존재하지 않는다

<br>

### 7.1.2. 단일 객체 연산과 다중 객체 연산

- 다중 객체 트랜잭션은 데이터의 여러 조각이 동기화된 상태로 유지돼야 할 때 필요하다
- 다중 객체 트랜잭션은 어떤 읽기 연산과 쓰기 연산이 동일한 트랜잭션에 속하는지 알아낼 수단이 있어야 한다
- 비관계형 DB는 연산을 묶는 방법이 없는 경우가 많다.
  - 어떤 키에 대한 연산은 성공하고 나머지는 실패해 부분적으로 갱신된 상태가 될 수 있다

**단일 객체 쓰기**

- 원자성과 격리성은 단일 객체를 변경하는 데에도 적용된다
  - 보내던 중간에 연결이 끊기면? 디스크가 전원이 나가면? 쓰는 도중이 읽기가 발생하면?
- 다루기가 무척 힘들기 때문에 저장소 엔진들은 거의 보편적으로 단일 객체 수준에서 원자성과 격리성을 제공하려 한다
- 단일 객체 연산은 갱신 손실을 방지해 유용하지만, 일반적으로 쓰이는 의미의 트랜잭션은 아니다(단일 객체이므로)

**다중 객체 트랜잭션의 필요성**

- 많은 분산 데이터스토어는 다중 객체 트랜잭션 지원을 포기했다. 근본적으로 가능은 하지만 넘 어려워서
- 단일 객체 트랜잭으로 충분치 않은 사례도 있다
  - RDB에서 다른 테이블의 로우를 참조하는 외래 키
  - 문서 데이터 모델에서 함께 갱신돼야 하는 필드들이 단일 객체로 다뤄지는 동일한 문서 내에 존재하는 경우
  - 보조 색인이 있는 DB에서 값을 변경할 때마다 색인도 갱신해야 한다

**오류와 어보트 처리 **

- 트랜잭션의 핵심 기능은 오류가 생기면 어보트되고 안전하게 재시도할 수 있다는 것
- 리더 없는 복제의 경우 DB는 최선을 다하지만 오류가 발생한다고 해서 이미 한 일을 취소하지는 않는다
- 레일스의 액티브레코드나 장고 같은 경우 어보트된 트랜잭션을 시도하지 않는다
- 어보트된 트랜잭션을 재시도하는 것은 간단하고 효과적이지만 완벽하지 않다
  - 커밋 성공을 알리는 중 네트워크가 끊긴다면
  - 오류가 과부하 때문이라면
  - 일시적이지 안혹 영구적인 오류라면
  - DB외부에도 부수적 효과가 있다면 (예컨대 e-mail 알림)
  - 클라이언트 프로세스가 재시도 중에 죽어버린다면

## 7.2. 완화된 격리 수준

- 두 트랜잭션이 동일한 데이터에 접근하지 않으면 안전하게 병렬실행될 수 있다
- 동시성 버그는 운이 없을 때만 촉발되기 때문에 테스트로 발견하기 어렵다
  - 매우 드물 수 있고 재현하기 어렵다
- 이때문에 DB들은 오랫동안 트랜잭션 격리를 제공해 동시성 문제를 감추려 했다
  - 직렬성 격리: DB가 여러 트랜잭션들이 직렬적으로 실행되는 것과 동일한 결과가 나오도록 보장하는 것
- 하지만 현실에서는 이게 무척 비싸기 때문에 일부 동시성 이슈만 보호해주는 완화된 격리 수준을 사용하는 것이 흔하다

### 7.2.1. 커밋 후 읽기

- 가장 기본적인 수준의  트랜잭션 격리
- 다음의 두 가질르 보장한다
  - DB에서 읽을 때 커밋된 데이터만 보게 한다 -> 더티 읽기 없음
  - DB에서 쓸 때 커밋된 데이터만 덮어쓰게 된다 -> 더티 쓰기 없음

**더티 읽기 방지**

- 더티 읽기: 아직 커밋되지 않은 데이터를 보는 것
- 더티 읽기를 막아야 하는 이유
  - 어떤 트랜잭션은 갱신된 값을, 나머지는 갱신되지 않은 값을 볼 수 있다 -> 사용자에게 혼란스럽다
  - 트랜잭션이 나중에 어보트 되어 롤백된 데이터, 즉 DB에 커밋되지 않을 데이터를 볼 수 있다

**더티 쓰기 방지**

- 더티 쓰기: 아직 커밋되지 않은 트랜잭션이 먼저 쓰고 나중에 실행된 쓰기 작업이 커밋되지 않은 값을 덮어쓰는 것
- 더티 쓰기를 막아야 하는 이유
  - 행동의 결과가 엉킨다: 한 아이템이 두 명에게 판매되는데 아이템은 A가 수령하고 지불은 B가 한다
  - 경쟁 조건을 막지 못한다 -> 뒤에서 자세히 다룸

**커밋 후 읽기 구현**

- 가장 흔한 방법으로, 로우 수준 잠금을 사용해 더티 쓰기를 방지한다
- 잠금으로 더티 읽기도 막을 수 있지만, 현실적으로 좋지 않다
- 현실에서 대부분의 DB는, 읽기 시에 커밋 이전의 과거의 값을 읽게 한다

### 7.2.2. 스냅숏 격리와 반복 읽기

- 커밋 후 읽기를 사용하더라도 트랜잭션 사이에 데이터가 꼬일 수 있다 (그림 7-6 참조) -> 비반복 읽기나 읽기 스큐라고 한다
- 이러한 비일관성은 일시적이지만, 때로 용납할 수 없을 때가 있다
  - 백업: 백업 프로세스는 오래 걸릴 수 있는데, 백업 중에 업데이트가 일어나면 데이터마다 버전이 달라진다
  - 분석 질의와 무결성 확인: 위와 비슷하게 데이터마다 버전이 달라진다
- 스냅숏 격리: 각 트랜잭션은 DB의 일관된 스냅숏으로부터 읽는다
  - 즉 트랜잭션은 시작할 때 DB에 커밋된 상태였던 모든 데이터를본다
  - postgresql, MySQL, 오라클, SQL 서버 등에서 사용

**스냅숏 격리 구현**

- 핵심 원리: 읽는 쪽에서 쓰는 쪽을 차단하지 않고, 쓰는 쪽에서 읽는 쪽을 차단하지 않는다는 것
- 진행 중인 여러 트랜잭션에서 서로 다른 시점의 DB 상태를 봐야할 수도 있기 때문에, DB는 객체마다 커밋된 버전 여러 개를 유지할 수 있어야 한다
  - 이를 다중 버전 동시성 제어MVCC라고 한다
  - 스냅숏 격리가 아니라 커밋 후 읽기 격리만 제공한다면 객체당 버전은 2개로 충분(커밋 버전, 덮어쓰였지만 커밋 안된 버전)

**일관된 스냅숏을 보는 가시성 규칙**

- 트랜잭션은 DB에서 객체를 읽을 때 트랜잭션 ID를 사용함으로써 볼 것과 안 볼 것들을 결정한다
- 동작 방식
  1. 각 트랜잭션을 시작할 때 그 시점에 진행 중인 모든 트랜잭션의 목록을 만든다. 이 트랜잭션들이 쓴 데이터는 모두 무시된다
  2. 어보트된 트랜잭션이 쓴 데이터는 모두 무시된다
  3. 트랜잭션 ID가 더 큰 트랜잭션이 쓴 데이터는 커밋 여부에 관계없이 모두 무시한다
  4. 그 밖의 모든 데이터는 애플리케이션의 질의로 볼 수 있다
- 다음 두 조건이 참이면 객체를 볼 수 있다
  - 읽기를 실행하는 트랜잭션이 시작한 시점에 읽기 대상 객체를 생성한 트랜잭션이 이미 커밋된 상태이다
  - 읽기 대상 객체가 삭제된 것으로 표시되지 않았거나 삭제요청 트랜잭션이 아직 커밋되지 않았다

**색인과 스냅숏 격리**

- 다중 버전 DB에서 색인이 동작하는 법
- 단순한 방법: 모든 버전에 색인이 있고 나중에 볼 수 없는 버전을 거른다
- postgresql의 경우 동일한 객체의 다른 버전들이 같은 페이지에 저장될 수 있다면 색인 갱신을 피한다
- 카우치DB, 데이토믹, LMDB에서는 추가 전용이며 쓸 때 복사되는 B트리의 변종을 사용한다
  - 트리의 페이지가 갱신될 때 덮어쓰는 대신 새로운 복사본 생성 그리고 자식 페이지들의 새 버전을 가리키도록 갱신
- 그러나 이 방법도 컴팩션과 GC를 실행하는 백그라운드 프로세스가 필요하다

**반복 읽기와 혼란스러운 이름**

- 스냅숏 격리는 유용해서 많이 쓰이는데, 이름이 다양하다
- 오라클에서는 직렬성, postgresql, MySQL에서는 반복 읽기라고 한다
- IBM DB2에서는 반복 읽기를 직렬성을 가리키는데 사용한다

### 7.2.3. 갱신 손실 방지

- 갱신 손실: 두 트랜잭션이 읽고 변경 후 다시 쓰는 작업을 할 때
  - 두 번째 쓰기가 첫 번째 변경을 포함하지 않아 하나는 손실 될 수 있다
- 무첫 흔한 문제라서 다양한 해결책이 있다

**원자적 쓰기 연산**

- 가능하다면 보통 가장 좋은 해결책
- 몽고는 JSON문서의 일부럴 지역적으로 변경하는 원자적 연산 제공
- 레디스는 우선순위 큐같은 데이터 구조를 변경하는 원자적 연산 제공
- 위키 페이지 갱신은 임의의 텍스트 편집을 포함해 연자적 연산으로 쉽게 표현되지 않는다
- 원자적 연산은 보통 그 객체에 독점적인 잠금을 획득해서 구현한다: 커서 안정성
  - 그 외에도 단일 스레드에서 실행되도록 강제할 수도 있다
- 객체 관계형 매핑 프레임워크를 사용하면 잘 안됨

**명시적 잠금**

- 앱이  읽기-수정-쓰기를 수행하는 중에 다른 트랜잭션이 오면 기다리도록 강제한다

**갱신 손실 자동 감지**



### 7.2.4. 쓰기 스큐와 팬텀



## 7.3. 직렬성

### 7.3.1. 실제적인 직렬 실행

### 7.3.2. 2단계 잠금(2PL)

### 7.3.3. 직렬성 스냅숏 격리(SSI)



























