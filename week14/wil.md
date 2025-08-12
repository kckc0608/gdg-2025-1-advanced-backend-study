Compaction 복습
disk 테이블이 점점 쌓임 → Read 비용이 증가함→  disk 테이블 수를 줄여주어야 함 

Compaction Strategy
- Leveled compaction
- Size-tiered compaction

최적의 Compaction Strategy 를 구현할 때 고려해야 하는 요소

1. 읽기 증폭 데이터를 읽을 때 여러 테이블에서 읽어와야 하는 것
2. 쓰기 증폭 compaction 과정에서 여러 번의 rewrite 가 발생하는 것
3. 공간 증폭 같은 key 에 대한 record 데이터를 여러 개 저장하는 것

그런데 어떤 특성을 개선하려고 하다보면 다른 특성이 나빠지는 경향이 존재함 
공간 증폭 개선
중복된 레코드 줄이기 → rewrite 횟수 증가 → 쓰기 증폭 증가 
쓰기 증폭 개선
rewrite 횟수 줄이기 → 중복된 레코드 증가 → 읽기 증폭, 공간 증폭 증가

다음 3가지 overhead를 고려하는 storage structure 에 대한 비용 모델

Read        (= 읽기 증폭)
Update   (= 쓰기 증폭)
Memory (= 공간 증폭)

latency, access pattern, 구현 복잡도, 유지보수 오버헤드, 하드웨어 특성 등을 고려하지 않은 모델이므로 완벽한 모델은 아니다.

RUM Conjecture 에 따르면 3가지 중에서 2가지를 개선하려고 하면, 나머지 한 가지가 안 좋아짐 (trade-off) 
B-Tree 
Read 에 최적화
Write 할 때마다 수정/삽입할 대상 데이터 위치를 찾아야 함 (write overhead)
미래에 쓸 데이터 크기를 고려해 여분 공간을 미리 확보해야 함 (space overhead) 
LSM Tree
write 할 위치를 찾을 필요도 없고, 미래의 write 를 고려한 여분 공간 확보도 필요하지 않음 (Write 최적화)
하지만 redundant record 를 저장하고 있음 (space overhead)
데이터 조회시 여러 disk table 에 접근해야 함 (read overhead)

LSM 트리에서 이 문제를 개선하기 위한 다양한 구현 기법들을 살펴보자

memory- / disk-resident table 을 구현하는 방법
secondary index가 동작하는 원리
Sorted String Table (disk-resident)
Skiplist (memory-resident)
Read 연산 수행 시 disk-resident table 접근 횟수를 줄이는 방법
Bloom Filters
Disk Access, Compression (2부)
log-storage structure 에 대한 새로운 아이디어
Unordered LSM Storage (2부)


disk-resident table 을 구현할 때 사용하는 자료구조
크게 index file 과 data file 로 구성
index file : B-Tree 나 hash table 같이 조회에 유리한 자료구조 활용
key 와 data entry 로 구성
data entry: data file 안에서 해당 key를 가진 레코드 위치의 file 시작점 기준 offset 값

data file : key 순서대로 데이터가 정렬되어 배치됨
hash table 을 써도 range scan 을 빠르게 할 수 있음 (시작점을 hash 로 찾고, 끝점에 도착할 때까지 데이터를 연속해서 탐색)
key-value 쌍이 연결된 (concatenated) 형태로 저장


Compaction 과정에서는
index 조회 탐색 없이 이미 정렬되어 있는 data file 을 순차적으로 조회
table merge 과정의 merge-iteration 과정도 순차적으로 data file 을 쓰면 되므로 single run 으로 해결 가능
file 이 가득 차면 immutable 로 간주되어 disk-resident content 는 수정되지 않음


LSM Tree 에서 쓰기 증폭이 발생하는 이유 → Read 수행 시 disk table 을 여러 개 조회해야 함 → why? search key를 어떤 disk table 이 갖고 있는지 알 수 없기 때문

Disk table 의 메타데이터로 보유하고 있는 key range(최소키, 최대키)를 저장한다면?
키 범위를 벗어난 테이블은 건너뛸 수 있겠지만, 여전히 이 정보로는 부족함
그 영역 안에 search key 가 존재할 ‘수'도 있다는 것만 알려주므로, 결국 테이블 내용을 읽어봐야 함
이 문제를 개선하는데 사용되는 자료구조가 Bloom Filter


공간 효율적인 확률 기반 자료구조
어떤 요소가 집합 안에 있는지 없는지 검사할 때 사용
확률 기반이기에 false-positive (실제로 없는데 있다고 판별) 케이스는 존재할 수 있음 하지만 false-negative (실제로 있는데 없다고 판별) 케이스는 절대 존재하지 않음  → 테이블(집합) 안에 key 가 “있을 수도 있다" or “확실히 없다" 를 알 수 있음       있다고 했으면 (positive) 진짜 있을 수도 있고 (True), 없을 수도 있는데 (False)      없다고 했으면 (negative) 진짜 없기만 하고 (True), 있는 경우는 없음 (False) → 확실히 없다는 것을 알 수 있음



Bloom Filter 에서 negative 가 나온 경우 → 해당 file 은 qeury 처리 시 건너뜀
Bloom Filter 에서 positive 가 나온 경우 → 실제로 record 가 있는지 file 을 탐색함


Q. 테이블 메타 데이터로 key range 저장하는 것도 똑같지 않나요?
Range 벗어난 key 값 → 해당 file 은 query 처리 시 건너뜀
Range 포함된 key 값 → 실제로 record 가 있는지 file 탐색

      → 성능에서 차이가 존재



Bloom Filter 는 Bit Array 와 여러 개의 해시 함수로 구성
찾고자 하는 record key 에 해시함수들을 각각 적용 → 해싱 결과는 Bit Array 의 인덱스 값들

만약 인덱스 값들이 가리키는 값이 모두 1 이면 → 이 테이블에 key 가 존재할 가능성 존재
만약 인덱스 값들이 가리키는 값에 0이 있으면 → 이 테이블에 key 는 확실히 없음



해시 함수 개수는 3개 (h1, h2, h3)
key1 의 경우
h1 ( key1 ) = 3
h2 ( key1 ) = 5
h3 ( key1 ) = 10

3개 인덱스가 가리키는 값이 모두 1 → key1 은 존재할 가능성이 있음


해시 함수 개수는 3개 (h1, h2, h3)
key4 의 경우
h1 ( key4 ) = 5
h2 ( key4 ) = 9
h3 ( key4 ) = 15

9, 15 가 가리키는 값이 0 → key4 는 존재하지 않음을 알 수 있음


인덱스가 가리키는 값이 모두 1 이어도 존재를 확신할 수 없는 이유 → 해시 충돌 가능성 (같은 함수로 서로 다른 key 를 해싱한 결과가 같은 것)

예시
key1 저장  → 3, 5, 10 인덱스의 bit 활성화
key2 저장 → 5, 8, 14 인덱스의 bit 활성화

key3 조회 → 3, 10, 14 인덱스가 모두 1
key3 이 저장된 것처럼 보임 (False Positive)
3, 10, 14 모두에 대한 해시 충돌이 원인
h1(key1) = h1(key3) = 3
h2(key1) = h2(key3) = 10
h3(key2) = h3(key3) = 14


Flase Positive 발생 가능성을 줄이는 방법
bit array 크기 키우기 (해시 충돌 가능성 낮추기)
bit array 크기가 늘어날수록 해시 충돌 가능성이 낮아짐
하지만 memory overhead 증가
hash function 개수 늘리기 (해시 충돌에 대한 저항성 높이기)
hash function 개수가 늘어날수록 해시 충돌이 그만큼 동시에 일어나야 하므로 FP 발생 확률이 낮아짐
하지만 연산 횟수가 늘어나므로 성능 부담 증가

→ FP 발생 가능성과 오버헤드 사이의 적절한 중간 타협점을 찾아야 함.
SSTable 은 불변성을 가지므로, 내부 원소 개수가 고정됨
이를 기반으로 이론적 최적 bit array 크기와 해시 함수 개수 계산 가능



in-memory 에서 데이터의 정렬을 유지하는 자료구조 중 하나
linked list 수준으로 구현이 간단하면서도, 확률적 복잡도는 search tree 와 비슷함
linked list 와 다르게 원소 삽입/수정시 rotation / relocation 대신 확률적 밸런싱 기법 사용
크기가 작아서 메모리 내 랜덤한 곳에 할당되므로 상대적으로 캐시 친화적이지 않다.


skiplist 는 서로 다른 높이를 갖는 node 의 나열로 구성
각 노드는 key 값과 포인터를 갖고 있음
linked list 와 달리 포인터가 여러 자식 노드를 가리킬 수 있음


높이가 h 인 노드는 최대 h 개의 노드를 이전 노드로 가질 수 있음
제일 낮은 레벨의 노드는 어떤 높이의 노드와도 모두 이어질 수 있다.


노드의 높이는 노드를 삽입할 때 random 함수에 의해 결정
같은 높이를 갖는 노드들은 level 을 구성   ex) level 1 = (3, 7, 15, 25), level 2 = (5, 22), level 3 = (10)
level 높이의 상한은 이 자료구조가 가질 수 있는 item 개수 기반으로 결정
level이 높아질수록 해당 level 을 구성하는 노드의 개수는 지수적으로 감소


제일 높은 level 부터 시작하여 탐색
현재 노드의 key < search key 이면 전진 현재 노드의 key > search key 이면 이전 노드로 돌아가서 다음 level 탐색
search key 또는 search key 의 이전 노드를 찾을 때까지 반복


7 탐색 과정 예시

제일 높은 레벨에서 시작
처음 만나는 노드는 10 > 7 이므로 전진을 멈추고 다음 레벨로 이어서 탐색

다음 레벨에서 처음 만나는 노드는 5 < 7 이므로 전진
다음 만나는 노드는 10 > 7 이므로 전진을 멈추고 이전 노드의 다음 레벨에서 이어서 탐색

key = 5 인 노드의 다음 레벨에서 탐색 시작
처음 만나는 노드의 키는 7 = 7 이므로 탐색 종료


삽입 위치를 조회 알고리즘과 동일한 방법으로 찾는다.
새로운 노드를 노드를 만든다.
tree와 유사한 계층 구조를 만들면서 균형을 유지하기 위해 확률 분포 기반의 random number 로 노드 높이 결정
앞 뒤 노드와 포인터를 연결한다.

높이가 3이고 key = 24 인 노드를 삽입한다면

key = 24 로 삽입 위치 탐색 → key = 22 인 노드에서 종료
key = 23 인 노드 생성 (랜덤하게 결정된 높이가 3이라고 가정)

높이가 3이고 key = 24 인 노드를 삽입한다면

key = 24 로 삽입 위치 탐색 → key = 22 인 노드에서 종료
key = 23 인 노드 생성 (랜덤하게 결정된 높이가 3이라고 가정)
link 앞 뒤로 연결


linked list 의 삭제처럼 point 만 변경
key = 23 인 노드를 삭제한다면, key = 23 인 노드의 forward link 가 가리키는 노드를 predecessor 노드의 포인터로 연결


fully_linked
fully_linked 속성을 추가, 포인터를 변경하고 있는 동안에는 false 로 설정하여 접근 방지
fully_linked 속성을 변경할 때는 CAS 방식을 사용하여 락 없이 안전하게 변경

reference count
C와 같이 메모리 관리 기능이 없는 언어를 사용한다면, 현재 노드가 참조 중인 개수 (reference count) 또는 hazard pointer 를 사용하여 현재 참조 중인 노드에 동시에 접근 중인 다른 스레드가 있는지 확인하고 메모리에서 해제
상위 레벨에서 아래 레벨로 일정한 흐름을 갖고 접근하기에 락 없이 동시성 제어 가능 