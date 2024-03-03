# 7장 - 분산 시스템을 위한 유일 ID 생성기 설계

하나의 관계형 데이터베이스를 사용할 경우 `auto_increment` 를 사용하여 ID 로 쓰지만, 여러 데이터베이스 서버를 쓰는 경우 지연 시간을 낮추기가 무척 힘들다.  따라서 `auto_increment` 가 아닌 다른 방법을 통해 ID 를 생성해야 한다.

## 유일 ID 생성기 설계 방법

### 다중 마스터 복제 (Multi-master Replication)

- 다중 마스터 복제는 데이터베이스의 `auto_increment` 기능 활용
- ID 값을 1만큼 증가시키는게 아닌 k(DB 서버의 수) 만큼 증가
- 장점
    - 데이터베이스 수를 늘리면 초당 생성 가능 ID 수가 늘기 때문에 규모 확정성 문제를 어느정도 해결할 수 있음
- 단점
    - 서버를 추가하거나 삭제할 때 잘 작동하도록 만들기 어려움
    - 여러 데이터 센터에 걸쳐 규모를 늘리기 어려움
    - 유일성은 보장되지만, 시간 흐름을 보장하는 것이 어려움

### UUID (Universally Unique Identifier)

- 유일성을 보장하는 ID 를 매우 간단히 만들 수 있는 방법
- 128 비트이며, 충돌가능성이 낮음
    - `123e4567-e89b-12d3-a456-426614174000`
    - 중복 UUID가 1개 생길 확률을 50%로 끓어 올리려면 초당 10억 개의 UUID를 100년 동안 계속해서 만들어져야 함
- 서버 간 조율 없이 독립적으로 생성 가능 → 동기화 이슈가 없음
- 시간순으로 정렬할 수 없음 → ULID로 대체 가능
- 숫자가 아닌 문자도 포함됨

### 티켓 서버 (Ticket Server)

- `auto_increment` 기능을 갖춘 데이터베이스 서버를 중앙 집중형으로 사용하는 방식

![Untitled](https://github.com/seongho-joo/Algorithm/assets/45463495/adb483bf-2b1a-4708-a185-c1564dce06db)

- `auto_increment` 를 활용해 키를 생성하기 때문에 유일성 보장하며 숫자로만 구성됨
- 구현하기 쉬움
- 티켓 서버는 SPOF(Single-Point-of-Failure) 임
    - 티켓 서버가 장애가 난다면, 해당 서버를 이용하는 모든 시스템이 영향을 받음
    - 이것을 해결하기 위해 여러 대를 둔다면 동기화 문제 발생

### 트위터 스노플레이크 (Snowflake)

- 트위터에서 공개한 키 생성 기법

![Untitled](https://github.com/seongho-joo/Algorithm/assets/45463495/d27cfe4e-71e9-4560-a0ef-1b89d7b6432e)

- 위 그림과 같이 여러 섹션으로 분할하여 사용
    - sign 비트 : 음수와 양수 구별
    - 타임스탬프 : epoch(기원 시각) 사용 / 비트를 가장 많이 차지함
    - 데이터센터 id : 2^n 만큼 데이터센터를 지원할 수 있음
    - 서버 id : 데이터센터당 2^n 만큼 서버를 사용할 수 있음
    - 일련번호 : 각 서버에서 키를 생성할 때마다 1씩 증가시킴 (1ms 가 경과할 때마다 0으로 초기화)
- 고려할 점
    - 시계 동기화(Clock Synchronization)
        - 각각의 서버는 시간을 동기화 해야함
        - 일반적으로 NTP(Network Time Protocol) 사용
    - section의 길이 최적화
        - 필요에 따라 각 부분의 비트를 조절할 수 있음
    - 고가용성(high availability)
        - 키 생성기는 필수 불가결(mission critical) 컴포넌트이므로 아주 높은 가용성을 제공해야 함

## 참고

- [Random UUID probability of duplicates](https://en.wikipedia.org/w/index.php?title=Universally_unique_identifier&oldid=755882275#Random_UUID_probability_of_duplicates)
- [ULID란 무엇일까](https://junuuu.tistory.com/823)
- [Clock synchronization](https://en.wikipedia.org/wiki/Clock_synchronization)
- [NTP 란](https://mindnet.tistory.com/entry/NTP)