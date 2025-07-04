# 4장 B-Tree 구현 (2)

## Rebalanciong
- 일부 구현에서는 노드를 쪼개거나 병합해야 할 때, 이 데이터를 형제노드로 보내거나 가져옴으로써 split / merge 연산 횟수를 줄이는 최적화 기법을 사용함.
- B*-tree 는 양쪽의 형제 노드가 꽉 찰 때까지 노드를 merge / split 하지 않는 방법으로 구현하고 있음.
- SQLite 는 노드를 쪼갤 때 형제노드까지 포함한 2개의 노드를 3개의 노드로 쪼개는 구현을 사용해서 merge / split 연산을 줄이는 최적화 기법을 활용함.
- 리밸런싱 구현은 스토리지 엔진의 복잡도를 높이지만, 이 기능은 기본적인 기능과 독립적이기 때문에, 별도의 stage 단계로 나눠서 구현하기도 한다.

## Right-Only Appends
- 많은 데이터베이스는 '항상 증가하기만 하는 값' 을 primary key 로 저장하는 경우가 있다.
- 즉, 새로 추가되는 데이터는 언제나 가장 마지막 데이터의 오른쪽이므로 split 연산은 항상 오른쪽에서 일어난다고 예상할 수 있다.
- Postgresql 에서는 이 케이스를 가리켜 fastpath 라고 부르며, 가장 오른쪽 노드 포인터를 갖고 있다가 삽입된 키가 가장 오른쪽 노드의 첫 번째 키보다 크다면 read path 를 거치지 않고, 가장 오른쪽 노드 안에 키를 삽입하는 방식으로 구현한다.
- SQLite 는 이를 quickbalance 라고 부르며, 새 키가 매우 오른쪽에 삽입되고, 목표 노드가 가득 차있다면 목표 노드에 대해 rebalancing 을 수행하지 않고 새로운 노드를 만들어서 채운다.

### Bulk Loading
- 사전에 정렬된 데이터에 대해 벌크 데이터를 가져오거나, 트리를 재구성한다면 Right-Only Appends 를 활용하기 좋다.
- 트리의 항상 오른쪽에 데이터를 삽입하기만 하고, split / merge 는 수행하지 않는 것이다.
- 채우는 과정에서 새로운 노드를 구성할 때 그 즉시 부모 노드도 채워나간다. (부모 노드도 split / merge 를 수행하지 않는다)
- 항상 대부분의 노드가 가득 차있기 때문에, 이 구성 방법은 비교적 빠르게 트리를 구성할 수 있는 방법이다.

## Compression
- 