# 4장 - 처리율 제한 장치의 설계

## 처리율 제한 장치(rate limiter)

- 클라이언트 또는 서비스가 보내는 트랙픽의 처리율을 제어하기 위한 장치
- ig) 특정 기간 내에 전송되는 클라이언트의 요청 횟수를 제한
    - 사용자는 초당 2회 이상 새 글을 올릴 수 없음
    - 같은 IP 주소로는 하루에 10개 이상의 계정을 생성할 수 없음
    - 같은 디바이스로는 주당 5회 이상 리워드를 요청할 수 없음

### 장점

- DoS 공격에 의한 자원 고갈(resource starvation) 방지
- 비용 절감
- 서버 과부화를 막음

## 설계 프로세스

### 문제 이해 및 설계 범위 확정

- 처리율 제한 장치를 구현하는 데는 여러 가지 알고리즘을 사용할 수 있음
- 면접관과 소통하면 어떤 제한 장치를 구현해야 하는지 분명히 할 수 있음

```
- 어떤 종류인지?
- 어떤 기준인지?
- 시스템 규모는 어느 정도인지?
- 분산 환경에서 동작하는지?
- 독립된 서비스인지?
- 사용자에게 알려줘야 하는지?
```

### 개략적 설계안 제시 및 동의 구하기

- rate limiter 를 어디에 둘 것인가?
    - 클라이언트측
        - 일반적으로 클라이언트는 위변조가 쉽게 가능하므로 안정적으로 걸 수 있는 장소가 못 됨
    - 서버측
        - API 서버
        - middleware
            - 마이크로서비스의 경우, API Gateway 컴포넌트에 구현됨
    - 지침
        - 기술 스택 점검 → 서버 측 구현을 지원하기 충분할 정도로 효율이 높은지 확인
        - 비즈니스에 맞는 알고리즘을 찾아라
        - 엔지니어링 인력
            - 적다면 상용 API Gateway 사용
            - 많다면 직접 구현
- rate limiter 알고리즘
    - [token bucket](https://en.wikipedia.org/wiki/Token_bucket)
    - [leaky bucket](https://en.wikipedia.org/wiki/Leaky_bucket)
    - [fixed window counter](https://etloveguitar.tistory.com/129)
    - [sliding window log](https://medium.com/@avocadi/rate-limiter-sliding-window-log-44acf1b411b9)
    - [sliding window counter](https://etloveguitar.tistory.com/130)
- 개략적인 아키텍처
    - 카운터는 메모리상에서 동작하는 캐시에 저장하는게 바람직 함

### 상세 설계

- 처리율 제한 규칙

    ```yaml
    # eg) 시스템이 처리할 수 있는 마케팅 메시지의 최대치를 하루 5개로 제한
    domain: messaging
    descriptors: 
      key: mesasge_type
    	Value: marketing
    	rate_limit:
    		unit: day
    		requests_per_unit: 5
    # eg) 클라이언트가 분당 5회 이상 로그인 할 수 없도록 제한
    domain: auth
    descritors:
    	key: auth_type
    	Value: login
    	rate_limit:
    		unit: minute
    		requests_per_unit: 5
    ```

- 처리율 한도 초과 트랙픽의 처리
    - HTTP 429 응답
        - 경우에 따라서 메시지 큐에 저장하여 나중에 처리
    - 처리율 제한 장치가 사용하는 HTTP 헤더
        - 클라이언트는 요청이 throttle 이 걸리고 있는지, 얼마나 많은 요청을 보내야 throttle 이 걸리는지 아래와 같은 헤더 정보를 보고 알 수 있음
        - X-Ratelimit-Remaining: 윈도 내에 남은 처리 가능 요청 수
        - X-Ratelimit-Limit: 매 윈도마다 클라이언트가 전송할 수 있는 요청 수
        - X-Ratelimit-Retry-After: 한도 제한에 걸리지 않으려면 몇 초 뒤에 요청을 다시 보내야 하는지 알림
- 분산 환경에서 처리율 제한 장치 구현
    - race condition
        - 락을 이용해 해결 가능하지만 성능을 상당히 떨어뜨림
        - 루아 스트립트 또는 정렬 집합을 사용
    - synchronization issue
        - 동기화는 분산 황경애서 고려해야 할 중요한 요소
        - 고정 세션 활용 → 규모면에서 확장 가능하지 않고 유연하지 않기 때문에 추천하지 않음
    - 성능 최적화
        - 에지 서버
        - 장치 간 동기화할 때 일광성 모델 사용
    - 모니터링
        - 다음과 같은 데이터를 확인하기 위해 모니터링을 함
            - 채택된 처리율 제한 알고리즘이 효과적이다
            - 정의한 처리율 제한 규칙이 효과적이다

### 마무리

시간이 남는다면 다음과 같은 부분을 언급하는게 좋음

- 경성 또는 연성 처리율 제한
    - 경성 처리율 제한: 요청 개수는 임계치를 절대 넘어설 수 없음
    - 연성 처리율 제한: 요청 개수는 잠시 동안은 인계치를 넘어설 수 있음
- 다양한 계층에서의 처리율 제한
    - 애플리케이션 계층을 제외한 다른 계층에서도 제한이 가능함
        - Ipatables를 사용하면 IP 주소 제한 적용 가능
- 회피 방법
    - 클라이언트 측 캐시를 사용하여 API 호출 횟수를 줄임
    - 임계치를 이해하고, 짧은 시간내에 많은 메시지를 보내지 않도록 함
    - 예외처리를 하여 우아하게 복구될 수 있도록 함
    - 지수 백오프 같은 백오프를 두어 retry 함