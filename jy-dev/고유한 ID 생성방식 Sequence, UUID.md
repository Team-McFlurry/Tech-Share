# 고유한 ID 생성 방식 Sequence, UUID

보편적으로 많이 사용하는 고유한 ID 생성방식인 Sequence와 UUID에 대해 설명하고자 합니다.

# Sequence

## 특징

- 해당 생성 방식은 가장 간단한 방식으로 int나 long타입 같은 정수형을 이용
- 4byte 혹은 8byte로 표현 가능
- 일반적인 케이스에서 순차적으로 증가하기 때문에 유추 가능

## 코드

```java
public class SequenceGenerator {
    private static long currentValue = 0;
    
    public synchronized static long getNextValue() {
        return ++currentValue;
    }
}
```

# UUID (RFC-4122)

## 특징

- UUID는 전역적(여러 다른 시스템, 다른 네트워크, 다른 환경)으로 고유한 값을 생성하기 위해 설계
- 랜덤의 32개의 16진수로 표현되면서, 5개의 그룹으로 나뉘어 하이픈으로 구분
- 16옥텟(1 옥텟 = 8비트) 128비트(16바이트) 구성
- 랜덤으로 생성되기 때문에 예측이 어려움
- 버전별로 구성이 조금씩 다름

## 생성 알고리즘

- MAC Address 사용
- 유사 난수 생성기 사용
- Application에서 제공한 문자를 암호화 해시를 통해 나온 결과의 일부 사용

## Layout And Byte Order

UUID의 일반적인 Layout은 아래와 같습니다. UUID 버전에 따라 각 필드별 구성되는 값은 다를 수 있습니다.

![Layout.png](https://github.com/user-attachments/assets/26d48769-cfed-494a-a2d7-49ee2bae4146)

### time_low (0~3 Octet)

- 32비트 정수로 구성
- 시간의 하위 비트

### time_mid (4~5 Octet)

- 16비트 정수로 구성
- 시간의 중위 비트

### time_hi_and_version (6~7 Octet)

- 16비트 정수로 구성
- 시간의 상위 비트와 버전 비트

### clock_seq_hi_and_reserved (8 Octet)

- 8비트 정수로 구성
- clock 시퀀스의 상위 비트와 예약된 비트

### clock_seq_low (9 Octet)

- 8비트 정수로 구성
- clock 시퀀스의 하위 비트

### node (10~15 Octet)

- 48비트 정수로 구성
- Node에 대한 식별 비트

## Version (4bit)

- time_hi_and_version의 상위 4bit
- 최대 16개의 버전 설정 가능

## Timestamp

- time_low(0 ~ 3 옥텟) + time_mid(4 ~ 5 옥텟) + time_hi_and_version(6~7 옥텟 Version bit 제외) =  60 bit로 구성
- UUID Version 1에서는 100 나노초 간격으로 계산된 UTC로 구성
- UUID Version 3, 5에서는 Name based로 구성
- UUID Version 4에서는 Random data로 구성

## Name based UUID

- Namespace ID(범주)와 Name(범주내의 고유 식별자)을 조합하여 생성
- 표준 Namespace ID (RFC 4122 정의)
    - DNS Namespace: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
    - URL Namespace: 6ba7b811-9dad-11d1-80b4-00c04fd430c8
    - OID Namespace: 6ba7b812-9dad-11d1-80b4-00c04fd430c8
    - X.500 DN Namespace: 6ba7b814-9dad-11d1-80b4-00c04fd430c8
- 구성 방식
    - Namespace ID (16바이트 UUID)를 네트워크 바이트 순서로 변환
    - Name을 바이트 시퀀스로 변환 (일반적으로 UTF-8 인코딩 사용)
    - Namespace ID와 Name 바이트를 연결
    - 연결된 바이트에 해시 함수 적용 (Version 3: MD5, Version 5: SHA-1)
    - 해시 결과의 bit를 사용하여 UUID 필드 구성
- 같은 Namespace ID와 Name 조합은 항상 동일한 UUID 생성

## Clock Sequence (16 bit)

- UUID Version1에서 Node Id나 시스템 시간이 역행되었을 때 중복방지를 위해 사용하는 필드
    - 만약 UUID Generator가 Stateless라면 해당 값은 랜덤으로 설정
- UUID Version3,5 에서는 해당 필드를 Name based의 값으로 설정
- UUID Version4 에서는 해당 필드를 랜덤으로 설정

## Variant(3bit)

![출처 :  RFC-4122 문서](https://github.com/user-attachments/assets/1bdfb166-cc5c-4e82-8199-e75fc9b5c861)

출처 :  RFC-4122 문서

- Variant는 다른 시스템에서 사용하는 고유 식별자에 대한 호환하기 위한 변형 비트
- clock_seq_hi_and_reserved(8 Octet)의 상위 3비트
- x로 표시된 비트는 아무값이 와도 상관 없음

### NCS 호환 예약 비트

- NCS는 1980년대에 개발된 분산 컴퓨팅 시스템
- 상위 1비트가 0으로 고정되는 경우 NCS 호환 UUID로 인식

### RFC-4122 표준 비트 (일반적으로 사용)

- RFC-4122에 나오는 정의대로 구현된 고유 식별자
- 상위 2비트가 1,0 순서대로 고정되는 경우 RFC-4122 표준 UUID로 인식

### Microsoft 호환 예약 비트

- Microsoft는 GUID(Globally Unique Identifier)라는 자체적인 고유 식별자을 사용
- 상위 3비트가 1,1,0 순서대로 고정되는 경우 GUID에 대한 호환 UUID로 인식

### 미래를 위한 예약 비트

- 1,1,1 순서대로 구성
- 사용하지 않음

## Node (48bit)

- UUID Version1 에서는 MAC 주소를 이용해 구성
    - 만약 여러 MAC 주소가 있는 경우에는 아무거나 사용할 수 있음
    - 이러한 MAC 주소가 없는 경우 Random으로 구성
        - Random시에는 MAC의 Multicast bit 1로 설정 필요 (충돌 확률을 줄이기 위해)
- UUID Version3,5 에서는 해당 필드를 Name based의 값으로 설정
- UUID Version4 에서는 해당 필드를 랜덤으로 설정

# Agenda

## Clustering Index에서의 사용?

## Another?