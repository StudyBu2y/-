# 5장 - 안정 해시 설계

> scale out 을 달성하기 위해서는 **요청 또는 데이터를 서버에 균등하게 나누는 것이 중요** →안정 해시 사용

## 해시 키 재배치 문제

- N개의 캐시 서버가 있다고 가정하고 이 서버들에 부하를 균등하게 나누는 보편적 방법은 다음 해시함수를 사용

```python
serverIndex = hash(key) % N # N은 서버 개수
```

- 캐시 서버 중 하나가 죽는다면 균등하게 분포된 데이터가 망가짐
    - cache miss 증가

## 안정 해시  (Consistent hash)

> **참고 영상**    
> https://www.youtube.com/watch?v=mPB2CZiAkKM&t=3598s    
> https://www.youtube.com/watch?v=1a4iG-SYWMc

- 해시 테이블 크기가 조정 될 때 평균적으로 오직 **k/n 개의 키만 재배치**하는 해시 기술 (k: 키 개수, n: 슬롯 개수)
- 해시 함수의 따라 해시 공간(hash space) 범위가 달라짐
    - 이러한 해시 공간을 구부려 해시 링(hash ring)이 만들어짐
  ![hash ring](https://miro.medium.com/v2/resize:fit:1060/0*eUVMkanQ4rsAOB43)
    - SHA-1 예시를 들면, 범위는 0부터 2¹⁶⁰ - 1 에 범위를 가지게 됨

```cpp
// Jump consistent hash
// Google 에서 공개한 해쉬 알고리즘
int32 _t JumpConsistentHash(uint64_t key, int32_t num_buckets) { 
	int64_t b =­-1, j = 0; 
	while (j < num_buckets) { 
		b = j; 
		key = key * 2862933555777941757ULL + 1; 
		j = (b + 1) * (double(1LL << 31) / double((key >> 33) + 1)); 
	} 
	return b; 
}
```

### 알고리즘 기본 절차

1. 서버와 키를 균등 분배(uniform distribution) 해시 함수를 사용해 해시 링에 배치
2. 키의 위치에서 링을 시계 방향으로 탐색하다 만나는 최초의 서버가 키가 저장될 서버

### 해시 키 배치

- 어떠한 키가 해시 공간에 배치되면, 시계 방향으로 탐색해 나가면서 만나는 첫 번째 서버에 저장됨
- 서버 추가 및 제거 시  추가 및 제거 되기전 서버 간 사이에 존재하는 키만 재배치 됨

### 기본 구현 법의 두 가지 문제

1. 서버가 추가되거나 삭제되는 상황을 감안하면 파티션의 크기를 균등하게 유지하는게 불가능함

   > 🤔 파티션(partition)   
    : 인접한 서버 사이의 해시 공간
    
2. 키의 균등 분포를 달성하기 어려움
    1. 다음 사진과 같이 키가 균등하게 배포되지 않기 때문에 핫스팟(hot spot) 키 문제가 발생할 수 있음

   ![hash ring2](https://www.baeldung.com/wp-content/uploads/sites/4/2023/05/hash-ring-01.png)


이 문제를 해결하기 위해 **가상 노드**(virtual node) 또는 **복제**(replica) 라 불리는 기법을 사용

### 가상 노드

- 가상 노드는 실제 노드 또는 서버를 가리키는 노드로서, 하나의 서버는 링 위에 여러 개의 가상 노드를 가질 수 있음
- 이 가상 노드의 개수를 늘릴 수록 표준편차가 작아져서 데이터가 고르게 분포되기 때문에 키의 분포는 점점 더 균등해짐

## 요약

### 안정 해시 이점

- 서버가 추가되거나 삭제될 때 재배치되는 키의 수가 최소화 됨
- 데이터가 보다 균등하게 분포하게 되므로 수평적 규모 확장성을 달성하기 쉬움
- 핫스팟 키 문제를 줄임

  >  🤔 hotspot 
    : 특정한 샤드(shard) 에 key 가 몰려 서버 과부하가 발생하는 문제

### 안정 해시를 사용하는 곳

- Amazon DynamoDB의 파티셔닝 관련 컴포넌트
- 아파치 카산드라 클러스터에서의 데이터 파티셔닝
- 디스코드 채팅 어플리케이션
- 아카마이 CDN
- 매그레프 네트워크 부하 분산기