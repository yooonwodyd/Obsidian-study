
```python
class UnionFind:
    def __init__(self, size):
        # 각 원소의 부모를 자기 자신으로 초기화
        self.parent = [i for i in range(size)]
        # 트리의 랭크(높이)를 저장하는 배열
        self.rank = [0] * size

    def find(self, x):
        # 원소 x의 대표 원소(루트)를 찾는 함수
        if self.parent[x] != x:
            # 경로 압축 최적화: 재귀적으로 부모를 찾아서 바로 연결
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x, y):
        # 원소 x와 y가 속한 집합을 합치는 함수
        xroot = self.find(x)
        yroot = self.find(y)

        if xroot == yroot:
            return  # 이미 같은 집합에 속해 있음

        # 랭크에 의한 합치기 최적화
        if self.rank[xroot] < self.rank[yroot]:
            self.parent[xroot] = yroot
        elif self.rank[xroot] > self.rank[yroot]:
            self.parent[yroot] = xroot
        else:
            self.parent[yroot] = xroot
            self.rank[xroot] += 1

```