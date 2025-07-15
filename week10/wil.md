동시성 제어 - 복습

동시에 실행 중인 트랜잭션들 사이의 상호작용을 다루는 기법
크게 3가지로 구분
낙관적 동시성 제어 (Optimistic Concurrency Control, OCC)
일단 실행하고, 커밋 전에 충돌을 검사하여 충돌이 있다면 충돌한 트랜잭션 중 하나를 골라 취소
다중 버전 동시성 제어 (MultiVersion Concurrency Control, MVCC)
하나의 레코드에 대해 여러 버전을 저장, OCC 처럼 일단 실행하고 커밋 전에 검증하는데, 검증 과정은 PCC 기법 (timestamp ordering, 2PL) 을 사용해 구현할 수 있다.
비관적 동시성 제어 (Pessimistic Concurrency Control, PCC) 
Lock-based  또는 nonlocking 기법을 사용하여 공유 자원에 대한 동시 접근을 막는 기법


낙관적 동시성 제어

트랜잭션 충돌은 거의 일어나지 않는다고 가정
락을 사용하여 트랜잭션 실행을 막는 대신, 실행 결과가 serializable 한지 검증하고 커밋하는 기법
각 트랜잭션을 실행하는 과정은 크게 3가지 단계로 구성
Read Phase
각 트랜잭션은 각자의 private context 에 대해 실행하고, 전체 트랜잭션의 read set / write set 파악
Validation Phase
동시에 실행된 트랜잭션의 read set / write set 에 대해 serializability 검증
검증에 실패했다면, 실패한 트랜잭션의 private context 초기화 후 Read Phase 부터 재시작
Write Phase
검증에 성공했다면 private context 에 있는 write set 을 database state 로 커밋

validation phase, write phase 는 원자적으로 수행 (항상 같이 실행되며, 중간에 다른 트랜잭션 간섭 불가)
validation phase, write phase 는 한번에 하나의 트랜잭션만 진행 가능
validation phase / write phase 는 read phase 에 비해 실행 시간이 매우 짧아서 성능 문제는 거의 없다.


검사 방법
backward-oriented : context 조회 이후에 커밋된 트랜잭션과 충돌이 발생했는지 검사
forward-oriented : validation phase 시점에 아직 커밋되지 않은 트랜잭션과 충돌이 발생했는지 검사


Backward-Oriented 동시성 제어는 모든 트랜잭션 쌍 T1, T2 에 대해 다음이 성립한다.
T1 은 T2의 read phase 가 시작되기 전에 commit 되어있다.
T1 은 T2의 write phase 가 시작되기 전에 commit 되어있고, T1 의 write set 과 T2 의 read set 은 서로소이다.
T1 은 T2의 read phase 가 시작되기 전에 read phase 가 끝난 상태 이다. T1 의 read / write set 과 T2 의 write set 은 서로소이다.

낙관적 동시성 제어는 검증이 대체로 성공하는 경우에 (충돌이 거의 없는 경우) 효율적
낙관적 동시성 제어는 임계 영역 이 존재 (한번에 하나의 트랜잭션만 접근할 수 있도록 제한해둔 영역 = V, W phase)
만약 일부 연산에 대해 ‘임계 영역' 을 만들지 않고 수행하고 싶다면 (nonexclusive ownership) 2부에서 알아볼 readers-writers lock 또는 upgradable lock 사용


레코드 버전을 여러 개 유지하고, 각 버전에 대응되는 transaction ID 또는 timestamp 를 활용하여 동시성 제어
각 버전은 트랜잭션의 커밋 여부에 따라 committed version / uncommitted version 로 구분
트랜잭션 매니저는 동시에 최대 1개의 uncommitted version 만 가지도록 동작

DBMS 에 설정된 isolation level 에 따라 uncommitted version 접근을 막기도 함
2PL 같은 lock 을 사용하는 기법, 또는 timestamp ordering 같은 lock-free 기법으로 구현 가능


읽기 연산 → 연산 시작 시점의 버전을 조회 → 모든 트랜잭션은 특정 과거 시점에 대해서 동일 데이터를 바라봄
쓰기 연산 → 새로운 버전 생성

충돌 검사는 커밋 시점에 검증 (OCC 와 유사)


읽기 연산 : 현재 트랜잭션의 시작 시간보다 뒤에 커밋된 버전의 데이터는 무시 (락은 사용하지 않음)
쓰기 연산 : 락 획득 → 새로운 버전 생성 (쓰기) → 락 해제 (2PL)

락을 활용하여 충돌을 제어하므로, 추가적인 충돌 검사는 하지 않음


각 버전에는 이 버전이 생성된 타임스탬프 값이 존재

읽기 연산 : 자신의 타임스탬프 값 이하의 버전 중 최신 버전 데이터를 읽음
쓰기 연산 : 쓰려는 대상의 가장 최신 타임스탬프가 자신보다 크면 abort, 작으면 update


동시성 제어 로직을 실행하는 동안 트랜잭션 충돌을 확인하고 트랜잭션의 실행을 중지시키는 기법
락 없이 제어하는 timestamp ordering
락으로 제어하는 2PL

timestamp ordering

각 트랜잭션은 timestamp 값을 보유
각 트랜잭션의 실행 여부는 자신보다 낮은 timestamp 값을 가진 트랜잭션의 커밋 여부로 결정

트랜잭션 매니저는 각 값(value)마다 이 값에 대한 최종 read / write 연산의 실행 시점을 나타내는 max_read_timestamp 와 max_write_timestamp 값을 관리 


읽기 연산 : max_read_timestamp 보다 낮은 timestamp 값을 읽으려고 하면 abort (새 버전의 데이터가 존재)
쓰기 연산 : max_read_timestamp 보다 낮은 timestamp 값에 대해 쓰려고 하면 abort                    max_write_timestamp 보다 낮은 timestamp 값에 대해 쓰는 것은 어차피 덮어쓸 것이므로 허용

읽기, 쓰기 연산을 수행한 뒤에는 해당하는 max_timestamp 값을 업데이트
aobrt 된 트랜잭션은 새로운 (더 높은) timestamp 값을 갖는 상태로 재시작 
이 규칙을 가리켜 Thomas Write Rule 이라고 부름


예시1
money = 10  (max_read_timestamp = 0, max_write_timestamp = 0)
 A 트랜잭션 시작 (timestamp = 10)
money 조회  → max_read_timestamp < 10 이므로 조회 성공 
money = 10 (max_read_timestamp = 10, max_write_timestamp = 0)

예시2
money = 10  (max_read_timestamp = 10, max_write_timestamp = 0)
 B 트랜잭션 시작 (timestamp = 5)
money 뒤늦게 조회  → max_read_timestamp > 5 이므로 조회 실패 → B 트랜잭션 abort 후 재시작 → 새로운 timestamp 값으로 (적어도 10보다는 큰 값) 시작  
money = 10 (max_read_timestamp = 10, max_write_timestamp = 0)


예시3
money = 10  (max_read_timestamp = 10, max_write_timestamp = 0)
 A 트랜잭션 시작 (timestamp = 15)
money 조회 후 업데이트  → max_read_timestamp < 15, max_write_timestamp < 15 이므로 읽기, 쓰기 성공 
money = 20 (max_read_timestamp = 15, max_write_timestamp = 15)


예시4
money = 10  (max_read_timestamp = 15, max_write_timestamp = 15)
 B 트랜잭션 시작 (timestamp = 10)
money 뒤늦게 업데이트 시도 → max_read_timestamp > 10 이지만, 쓰기 동작을 abort 하지는 않음 → 다만 timestamp 10 시점에서 쓰는 값은 어차피 15 시점에서 쓰는 값에 의해 덮여질 값이었으므로 무시


2PL

2-Phase Locking
명시적인 Lock 을 활용한 대표적인 비관적 동시성 제어 기법
이름대로, 락 관리를 2단계로 나누어서 수행
the growing phase (expanding phase) 트랜잭션이 필요로하는 모든 락을 획득하기만 하는 단계
the shrink phase growing phase 에서 획득한 모든 락을 해제하는 단계

즉, 트랜잭션은 필요한 락을 획득하기만 하거나, 기존 락을 모두 해제하거나 2가지 선택지만 수행 가능

예시 -- Growing Phase --
 lock(x)      ← 락 획득
 read(x)
 lock(y)      ← 락 획득
 write(y)

  -- Shrinking Phase --
 unlock(x)    ← 락 해제
 unlock(y)    ← 락 해제


데드락

Locking protocol 에서 락을 획득하려면, 먼저 그 락이 해제되기를 기다렸다가 획득해야 함.
데드락 = 두 개의 트랜잭션이 서로가 서로의 락을 필요로 하면서 둘 다 무한히 기다리는 상황

무한히 기다려야된다? → 일정 시간 동안 트랜잭션이 동작을 안하면 데드락으로 간주하고 abort 시키기
conservative 2PL 기존 2PL 보다 더 엄격한 방식 → 트랜잭션이 동작을 시작하려면, 전체 동작 수행에 필요한 모든 락을 사전에 획득해야 동작 시작 가능      만약 불가능하면 abort


→ 둘 다 시스템의 동시성 제어 능력을 제한하는, 성능상 별로 안 좋은 기법      DBMS 는 트랜잭션 매니저를 활용하여 데드락을 감지하거나, 회피(예방)한다.


감지

waits-for graph 를 활용
실행중인 트랜잭션들간 락 의존 관계를 추적 → 사이클이 존재하면 데드락 존재
주기적으로 검사 or waits-for graph 가 업데이트 될 때마다 검사 → 가장 최근 락을 획득한 트랜잭션을 abort


회피

트랜잭션 timestamp 를 활용하여 우선순위 부여
낮은 timestamp (시간이 오래된 경우) → 높은 우선순위

T1 이 락을 먼저 획득한 T2 의 락을 갖고자 할 때
wait-die T1 의 timestamp 가 작다면 (=높은 우선순위) block 될 수 있다. (락이 해제 되기를 기다릴 수 있다) T1 의 timestamp 가 크다면 T1은 abort 되고 다시 시작한다.
Wound-wait T1 의 timestamp 가 작다면 (=높은 우선순위) T2 가 abort 되고 재시작한다. (wound) T1 의 timestamp 가 크다면, 락을 기다릴 수 있다. 


정리

트랜잭션 처리를 위해서는 데드락 관리를 위한 스케줄러가 필요
반면 Latch 는 데드락 회피 기법 대신, 프로그래머가 데드락이 발생되지 않게 락 순서를 관리하도록 함.

2부에서 더 자세히 알아봅니다