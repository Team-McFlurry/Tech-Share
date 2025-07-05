# Spring Data x Redis 병목 시나리오



## 1. 개요

- Spring Data 모듈에서 제공하는`CrudRepository`를 이용하면 DB 종류에 상관없이 같은 인터페이스로 CRUD 작업을 손쉽게 할 수 있다.
- Redis를 사용할 때는 엔티티 클래스만 정의한 뒤 `CrudRepository`를 상속하는 도메인 `Repository` 인터페이스를 선언하기만 하면,
  런타임에 스프링이 자동으로 적절한 구현체 빈을 생성해 준다.
- Redis를 사용할 때 스프링에서 제공하는 실제 구현체는 `SimpleKeyValueRepository`이며, 이 클래스가 Redis에 명령을 전달한다.



## 2. 문제 상황 – 다중 요청

대부분의 상황에서 Spring Data 모듈을 통해 Redis를 사용해도 성능 차이는 거의 없다.

그러나 여러 요청을 한 번에 처리해야 하는 경우에는 Spring Data를 이용하면 성능이 크게 떨어질 위험이 있다.



다중 키 조회 상황을 생각해보자.

`CrudRepository`가 `findById()`와 `findAllById()`를 별도로 제공하므로,

`findAllById()`가 내부적으로 최적화를 해 줄 것처럼 보일 수 있다.



하지만 실제 구현체인 `SimpleKeyValueRepository`의 `findAllById()` 구현은 다음과 같다.

```java
public List<T> findAllById(Iterable<ID> ids) {
        Assert.notNull(ids, "The given Iterable of id's must not be null");
        List<T> result = new ArrayList();
        ids.forEach((id) -> {
            Optional var10000 = this.findById(id);
            Objects.requireNonNull(result);
            var10000.ifPresent(result::add);
        });
        return result;
    }
```

- 키 수(N)만큼 `findById`를 반복 호출하며, 결과를 리스트로 모아 반환한다.
- 각 호출마다 **클라이언트 ↔ 서버** 왕복이 한 번 발생한다.



아무리 고성능 `key-value store`인 Redis라도 **TCP 기반 클라이언트–서버 모델**을 따르는 이상, I/O 병목에 노출될 수 있다.



`READ` 명령 5개를 연달아 보낸 상황을 가정해 보자.

1. 클라이언트가 첫 번째 요청을 전송하고 응답을 기다린다.
2. 응답을 받은 뒤 두 번째 요청을 보낸다.
3. 같은 흐름을 총 다섯 번 반복한다.



이처럼 **다섯 차례 요청–응답을 순차로 처리**하면 RTT가 그대로 다섯 배 누적된다.

요청 수가 늘어날수록 지연도 선형적으로 커지기 때문에 서버 전체 처리량과 응답 시간이 급격히 악화될 수 있다.



## 3. Redis가 제공하는 두 가지 해법

### 3-1. MGET – 여러 스트링 키를 한 번에 읽기

`MGET key1 key2 … keyN` 명령은 여러 키를 한 요청으로 받아 한 응답에 담아준다.

```kotlin
val values: List<String>? = redisTemplate.opsForValue().multiGet(keys)
```

그러나 키에 매핑된 값의 타입 Hash인 경우 `MGET`을 사용할 수 없다.

그래서 보다 범용적인 방법으로는 레디스 파이프라인을 사용하는 것이 좋다.



### 3-2. 파이프라이닝 – 여러 명령을 한 커넥션으로 묶기

#### 레디스 파이프라인이란?

- 이전 요청의 응답을 기다리지 않고 새로운 요청을 보낼 수 있는 기능
- 실제로는 파이프라인으로 전송한 클라이언트의 요청을 모아서 Redis로 한번에 flush한다.
- 여러 명령어를 동시에 송신하여 네트워크 RTT를 절약할 수 있다



해시·리스트·정렬집합처럼 `MGET`에 해당하는 단일 명령이 없을 때는 파이프라이닝을 이용해야만 한다.

클라이언트는 요청 N개를 한꺼번에 보내고, 응답 N개를 한꺼번에 받으므로 RTT를 크게 감소시킬 수 있다.

```kotlin
// 해시 연산 준비
val hashOps = redisTemplate.opsForHash<String, String>()

// 파이프라인 실행
val pipelineResult: List<Any> = redisTemplate.executePipelined(object : SessionCallback<Any> {
    override fun <K, V> execute(operations: RedisOperations<K, V>): List<Any>? {
        // 여러 해시 키에 대해 entries(key)를 파이프라인에 쌓는다
        keys.forEach { key ->
            hashOps.entries(key)
        }
        // 콜백 내부에서는 return emptyList()
        return null
    }
})
```



## 4. 실험 – 서비스 적용 전후 성능 비교

- 조회 키 수 (N) : 120
- 데이터 타입 : Hash

| 구분    | 방식                           | 평균 응답 시간 (ms) |
| ------- | ------------------------------ | ------------------- |
| 변경 전 | `CrudRepository.findAllById()` | 320                 |
| 변경 후 | redis pipelining               | 25                  |

