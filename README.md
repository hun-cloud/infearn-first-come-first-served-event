# [인프런] 실습으로 배우는 선착순 이벤트 시스템

## 강의 소개
선착순 이벤트 쿠폰을 100장을 생성하여 발급할 때 발생할 수있는
문제점과 해결 방법

## 발생할 수 있는 문제점
- 쿠폰이 100개보다 많이 발급될 수 있다.
- 이벤트 페이지 접속이 안될 수있다.
- 이벤트와 전혀 상관 없는 페이지도 느려질 수 있다.

## 문제 해결
- 트래픽이 몰렸을 때 대처할 수 있는 방법을 배운다.
- redis를 활용하여 쿠폰 발급개수를 보장한다.
- kafka를 활용하여 다른 페이지들에 대한 영향도를 줄인다.

## 다루지 않는 것
- coupon 도메인
------------

# 정리

### 요구사항 정의
```agsl
선착순 100명에게 할인쿠폰을 제공하는 이벤트를 제공한다.

이 이벤트는 아래와 같은 조건을 만족하여야 한다.
- 선착순 100명에게만 지급되어야 한다.
- 101개 이상이 지급되면 안된다.
- 순간적으로 몰리는 트래픽을 버틸 수 있어야 한다.
```
### 100개의 쿠폰을 발급해주는 로직의 문제

```java
    @Test
    void 여려명_응모() throws InterruptedException {
        int threadCount = 1000;

        ExecutorService executorService = Executors.newFixedThreadPool(32); 
        CountDownLatch latch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            long userId = i;
            executorService.submit(() -> {
                try {
                    applyService.apply(userId);
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();

        long count = couponRepository.count();

        assertThat(count).isEqualTo(100); // false 쿠폰이 110개가 생김
    }
```
멀티 스레드로 1000개의 요청을 하였을 때 테스트 케이스가 실패하는 것을 확인할 수 있다.
쿠폰이 100개 이상이 발급되버리는 문제가 발생한다.

### 레이스 컨디션

두 개 이상의 스레드가 공유 자원을 서로 사용하려고 경합하는 현상을 의미한다.

### 레디스를 사용하여 해결해보자

레디스의 incr을 사용하여 coupon 갯수를 101개 이상을 생성하지 못하도록
제한할 수 있다. 이것이 가능한 이유는 레디스는 싱글 스레드를 기반으로 하기 때문이다.

### 레디스를 사용하였을 때의 장점
1. DB의 락을 걸었을 때 보다 성능적 이점을 가져올 수 있다.
2. 101개 이상의 쿠폰이 발급되지 않는다.


### 레디스만 사용했을 때의 문제점
발급하는 쿠폰의 갯수가 많아지면 많아질수록 RDB에 부하가 오게 된다. 사용하는 DB가
쿠폰 전용 DB가 아닌 여러 곳에서 사용하는 DB라면 다른 서비스의 장애에도 큰 영향을 주게 된다.

결론적으로 쿠폰 갯수가 많아지면 많아질수록 RDB 리소스의 사용량이 증가하게 되고 이것은 장애를 발생시키게 된다.

### 카프카를 사용하여 해결해보자
카프카는 분산 이벤트 스트리밍 플랫폼이다.
이벤트 스트리밍이란 소스에서 목적지까지 이벤트를 실시간으로 스트리밍 하는 것이다.

### 카프카를 사용하여 처리 시 장점
- api 처리량을 조절할 수 있음
- DB 부하를 줄일 수 있음

ButTest case에서 확인했듯이 약간의 딜레이가 발생할 수 있음

## 발급 가능한 쿠폰 1인당 1개로 제한하기

방법 1 : 데이터베이스 레벨에서 복합키를 걸어서 막기
-> 실용적인 방법이 아님

방법 2 : 범위로 lock을 걸어 쿠폰 발급 여부를 판단하여 발급 
-> 쿠폰 발급이 되기 전에 또 요청한다면 동일한 사용자가 쿠폰을 2개 발급 받을 수 있음

### 방법3 : redis의 set 자료구조 사용하기

sadd 사용하여 redis에 set을 데이터를 추가할 수 있다.
신규 추가라면 1를 리턴, 이것을 코드에 적용하여 1인당 1개의 쿠폰을 발급할 수 있다.















