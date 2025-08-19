# LSM Tree 에서의 동시성 제어
LSM 트리의 동시성 제어 이슈는 크게 table view 변경과 로그 동기화이다.
Memtable 도 동시 접근이 되지만, 메모리에서의 동시성 제어는 다루지 않음

flush 과정은 다음과 같음

1. 새로운 memtable 에 대해 읽고 쓰기가 가능하게 만듦
2. 기존 memtable 은 읽기만 가능하게 만듦
3. 기존 memtable 을 디스크로 flush
4. flush 가 끝나면 기존 memtable 제거하고, 디스크의 memtable 로 대체, 이 과정은 원자적으로 수행됨
5. 관련된 WAL 로그를 제거

일반화해보면 다음 요소를 고려해야 함

- Memtable switch
- Flush finalization
- Write-ahead log truncation

이 요소를 고려하지 않으면 정합성 문제가 발생할 수 있음.
예를 들어 과거 memtable 에 대해 계속 쓰기를 진행해서, 거기에 쓴 데이터는 유실될 수 있음
compaction 과정에서도 테이블 뷰가 변경될 수 있지만, 조금 더 단순함.
이전의 디스크 테이블은 폐기하고, 새로운 compaction 된 테이블이 추가 됨.
다만 이 과정에서 하나의 테이블에 여러 compaction 에 동시에 참여하지 않도록 해야함.

LSM 트리도, B-Tree 처럼 로그를 자를 때 페이지 캐시의 더티 페이지를 디스크로 플러시하는 과정과 반드시 동기화해야 함.
만약 잘라낸 로그 데이터와 플러시된 데이터가 다르면 데이터 손실이 발생할 수 있다.

# Log Stacking
많은 현대 파일 시스템은 로그 구조로 되어있어, 메모리 세그먼트에 쓰기를 버퍼링하고, 세그먼트가 가득 차면 디스크에 append-only 방식으로 내보내는 구조를 따름.
이를 통해 소규모 랜덤 쓰기를 줄여 쓰기 오버헤드를 줄이고, wear leveling 을 개선하여 장치 수명을 늘림.

특히 LSM 과 SSD 는 그 조합이 좋음.
SSD 는 in-place 업데이트가 성능 저하를 일으키는데, 순차적 워크로드와 append-only 조합은 SSD 가 유리하기 때문.
하지만 로그 구조 시스템을 겹겹히 쌓으면 LSS로 풀려던 문제가 재발할 수 있음.
따라서 어플리케이션을 개발할 때는 FTL 을 고려해야 함.

SSD 에서 SSL 를 쓰기 좋은 이유는 크게 2가지

1. 작은 랜덤 쓰기를 물리적  페이지 단위로 모아서 처리할 수 있고
2. SSD 가 program(write)/erase(delete) 사이클로 동작하기 때문

쓰기는 오직 빈 페이지에서만 가능하다.
하지만 페이지를 비우는 것은 여러 페이지를 묶은 블록 단위로만 할 수 있음

FTL 이 빈 페이지를 다 쓰면, 가비지 컬렉션을 실행해서 폐기된 페이지가 들어있는 블록을 지우고, 그 안에 살아있는 페이지는 다른 블록에 옮김.
I/O 를 묶어서 처리하면 그만큼 가비지 컬렉션 빈도가 줄면서 추가적으로 I/O 가 감소하여 SSD 수명이 증가함

# 파일 시스템
SSD 위에는 파일시스템이 존재함.
파일 시스템들 역시 로그 기법으로 쓰기를 버퍼링함.
이를 통해 쓰기 증폭을 줄이고, 하드웨어 최적화된 방식으로 활용이 가능해짐.

이렇게 로그 시스템이 겹겹히 쌓인 것을 가리켜 Log Stacking 이라고 함.
근데 이건 몇가지 문제를 일으킬 수 있음.

1. 각 계층은 자체적으로 기록 관리를 해야 함. 하위 계층이 자신의 로깅 정보를 상위에 전달하지 않는 경우가 대부분이라, 하위 로그와 중복적인 로그를 남기는 문제가 발생할 수 있음.
이로 인해 어플리케이션 계층과 파일 시스템 계층에서 중복된 로깅과 서로 다른 가비지 컬렉션 패턴이 발생할 수 있음.

2. 세그먼트 쓰기 불일치도 문제임. 상위 계층 로그의 세그먼트를 폐기할 때, 이웃 세그먼트가 단편화되거나 다시 옮겨지면서 세그먼트가 불일치함.
이로 인해 상위 계층의 세그먼트가 하위 계층의 세그먼트 크기와 맞지 않아 여러 세그먼트를 차지하는 문제가 발생할 수 있음.

근데 이런 문제는 줄이거나 피할 수 있음.
흔히 LSS 가 순차 I/O 에 대한 것이라고 생각하기 쉬운데, 실제 DBMS 는 다중 쓰기를 하고 있따는 점을 염두에두어야함.


# mindful stacking
Bw-트리는 기본 노드(base node)와 수정 사항(delta node)을 분리해서 기록한다. 
기본 노드와 델타 노드는 체인 형태(링크드 리스트)로 구현된다.
델타 노드(delta nodes)는 삽입, 업데이트(삽입과 구별할 수 없음), 또는 삭제를 나타낼 수 있다.

하지만 델타 노드를 무한히 연결할 수는 없기 때문에, 새로운 하나의 노드로 합치는 과정이 필요
새로운 노드는 디스크의 새 위치에 기록되고, 매핑 테이블의 노드 포인터는 이를 가리키도록 업데이트된다. 
이 프로세스에 대해서는 "LLAMA and Mindful Stacking"에서 더 자세히 논의한다.  이는 log-structered storage 가 가비지 컬렉션, 노드 통합 및 재배치를 담당하기 때문이다.

Bw-Tree 는 LLAMA(Latch-free, Log-structured, Access-Method aware) 위에서 구현
      → Bw-Tree가 필요로 하는 GC와 페이지 관리는 LLAMA 를 이용            이런 계층화 덕분에 Bw-Tree 가 동적으로 늘어나고 줄어들도록 구현 가능 
LLAMA 의 ‘Access-Method aware’ 파트가 software 계층과 어떻게 조합해서 쓰이는지 Bw-Tree 와 LLAMA 사례를 통해 알아보자

Log-structured Storage (LSS) 은
레코드 수정 사항을 flush buffer 에 모아두었다가, 페이지가 가득차면 한번에 디스크로 flush 함
주기적으로 GC 를 수행하여 현재 디스크에서 사용하지 않는 delta node, base node 를 회수하고(reclaim) 사용중인(live) 노드를 한 곳에 모아 단편화된 페이지를 제거(relocate)

만약 access-method 를 고려하지 않는다면..
서로 다른 logical node 에 대한 delta node 들이 삽입 순서대로 뒤섞여 있을 때, 이들을 하나로 모으는 과정에서 그냥 삽입 순서 그대로 모아버리게 됨

LLAMA 는 Bw-Tree 의 구조를 알고 있기 때문에 access-method 를 고려 (awareness)
하나의 논리 노드에 대한 여러 delta node 를 모아서 하나의 연속적인 물리 공간에 저장

LLAMA 는 Bw-Tree 의 구조를 알고 있기 때문에 access-method 를 고려 (awareness)
하나의 논리 노드에 대한 여러 delta node 를 모아서 하나의 연속적인 물리 공간에 저장
만약 두 delta node 의 내용이 서로 충돌한다면, 두 delta node 를 논리적으로 병합 (레코드를 생성 후 삭제 → 레코드의 삭제만 최종 저장)

LLAMA 는 Bw-Tree 의 구조를 알고 있기 때문에 access-method 를 고려 (awareness)
하나의 논리 노드에 대한 여러 delta node 를 모아서 하나의 연속적인 물리 공간에 저장
만약 두 delta node 의 내용이 서로 충돌한다면, 두 delta node 를 논리적으로 병합 (레코드를 생성 후 삭제 → 레코드의 삭제만 최종 저장)
GC 는 Bw-Tree node 내용의 논리적인 통합까지도 처리할 수 있다. (변경점을 모아 하나의 노드로 합치는 일) → 여유 공간을 모으는 일 + 물리 node의 fragment 를 줄이는 일 + read 레이턴시 감소


→ Bw-Tree 와 LLAMA 의 결합으로 인해 GC로 하여금 단순한 ‘공간 회수’ 이상의 의미를 가지게 만들어줌

이처럼 신중하게 고려해서(mindful) 레이어를 쌓으면(stacking) 계층적 설계로 많은 이점을 얻을 수 있음
       (mindful = 각 계층이 무슨 역할을 해서 다른 레이어에 어떤 긍정적인 효과를 주는지 고려하는 것)

그렇다고 항상 레이어를 쌓아서, 레이어 간 강한 결합을 이루는 설계를 해야 하는 것은 아님
API 를 잘 설계해서 올바른 정보를 잘 노출하면, 다수의 레이어를 쌓지 않아도 성능을 크게 개선할 수 있음
      Ex) 소프트웨어와 하드웨어 사이의 간접 레이어를 생략하고, 하드웨어에 직접 접근해서 데이터를 다루는 방식

# Open-Channel SSD

Open-Channel SSD 를 통해 개발하면 파일 시스템과 FTL(flash translation layer) 을 생략할 수 있음
최소한 2개 계층에 대한 로깅 생략 가능 (파일 시스템, FTL)
wear-leveling, GC, data placement, 스케줄링에 대한 제어권 증가

구현 예시 : LOCS, LightNVM

FTL 은 주로 data placement, GC, page relocation 담당
Open-Channel SSD 에서는 드라이브 관리, I/O 스케쥴링 기능을 FTL 없이 다룰 수 있도록 외부에 노출
개발자 관점에서 디테일을 모두 신경써야 하지만, 큰 성능 향상을 가져올 수 있음

SDF (Software Defined Flash)
소프트웨어, 하드웨어 모두 고려하여 설계된 Open-Channel SSD
SSD 의 특성을 고려하여 비대칭적인 I/O 인터페이스 제공
read unit 크기 ≠ write unit 크기
write unit 크기 = erase unit(=block) 크기
쓰기 증폭 감소에 효과적(크게 지우고, 작게 여러 번 쓰지 않아도 됨)
LSS 에도 효과적 → 1개 계층만으로 GC, page relocation 을 수행 가능
SSD 병렬 접근 제어도 가능 → SDF 의 각 채널은 개별 block device 에 노출 → 이를 잘 활용하면 성능을 더욱 개선 가능

복잡한 내부 동작을 숨기고 간단한 API 만 공개하는 것은 꽤 좋아보임
하지만 서로 다른 의미를 가지는 소프트웨어 계층을 통합하려고 하면, 복잡한 문제를 일으킬 수 있음
이럴 때는 오히려 아래 계층의 내부 구조를 노출시키는 것이 계층 통합에 더 유리할 수도 있음

# 6장 요약

Log-structured storage (LSS)
flash transition layer, file system, database system 등 여러 곳에서 활용
작은 규모의 random write 연산을 메모리에 모았다가 한번에 수행하여 쓰기 증폭 감소
주기적으로 가비지 콜렉션을 수행하여 삭제된 세그먼트가 차지하는 공간 회수

LSM Tree
LSS 아이디어를 차용하여 로그 구조 기반으로 관리되는 인덱스 구조를 만드는데 활용
쓰기 연산을 메모리에 모았다가 디스크로 한번에 flush
과거 버전의 record 는 compaction 과정에서 정리됨

Log Stacking & Open-Channel SSD
많은 소프트웨어 레이어는 LSS 를 활용, 각 레이어들을 최적의 형태로 쌓는 것이 중요 (mindful stacking)
경우에 따라서 파일 시스템 계층을 생략하고 하드웨어에 직접 접근하는 것도 방법도 존재 (Open-Channel SSD)


# 파트 1 요약

데이터베이스와 스토리지 엔진
데이터베이스 아키텍처와 큰 분류 (OLTP, OLAP, HTAP)
디스크 기반 스토리지 구조는 어떻게 구현하는지 다른 요소와는 어떻게 상호작용하는지

B-Tree
기본적인 B-Tree, B-Tree 구현, B-Tree 변형체
buffering, immutability, ordering 특성을 중심으로 다양한 스토리지 엔진의 특징을 살펴봄

Buffering
메모리 버퍼를 추가하면, 쓰기 증폭 문제를 개선할 수 있음
in-place update 특성을 가진 스토리지 구조는 in-memory buffer를 통해 같은 페이지에 대한 여러 쓰기 연산을 하나의 쓰기 연산으로 만들어 처리함
WiredTiger, LA-Tree

Immutability
불변 스토리지 구조(multicomponent LSM Tree, FD-Tree) 역시 buffering 을 쓰면 쓰기 증폭이 줄어듦
다만 immutable level 사이에서 데이터를 이동하는 경우,  다수의 쓰기 연산이 발생하므로 지연된 (deferred) 쓰기 증폭 문제가 발생함
그래도 공간 증폭 문제, 동시성 제어 관점에서는 불변 스토리지가 유리함. (대부분의 불변 스토리지는 완전히 페이지를 다 채워서 쓰기 때문)

Ordering
immutability 특성을 사용해도, buffering 이 없다면 결국 unordered 스토리지 구조가 됨 (Bitcask, WiscKey)
다만 CoW B-Tree 는 페이지를 복사할 때 정렬, 재배치를 수행하므로 예외적으로 정렬성 유지
WiscKey 는 key만 정렬된 LSM Tree 내에 저장, 정렬된 key index로 비정렬 레코드 조회
Bw-Tree 는 노드의 일부만 key 순서대로 정렬해서 저장, 나머지 노드들은 여러 페이지에 흩어져있는 delta node 에 변경점 저장

Buffering, Immutability, Ordering 속성을 적절히 조합하면, 원하는 특성을 갖는 스토리지 엔진 설계 가능
하지만 이 과정에는 trade-off 가 존재
이 특성을 이해하면 현대 DBMS 코드를 더 자세히 이해할 수 있고, 학습이 쉬워짐

많은 현대 DBMS 들은 확률적 데이터 구조 기반으로 구동
최근에는 DBMS에 기계학습을 접목하는 연구 진행 중
nonvolatile byte-addressable 스토리지 보급으로 연구도 계속 변화하는 중

이 책에서 소개된 기본적인 개념들을 이해하면 새로운 연구를 이해하고 구현하는데 도움이 됨.
새로운 개념들 역시 기존 개념들 기반으로 만들어지기 때문

