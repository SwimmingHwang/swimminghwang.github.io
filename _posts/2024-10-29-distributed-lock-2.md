---
layout: post
title: Redis와 락 (feat. redisson)
author: Soo-young Hwang
tags:
  - redis
category: redis
published: true
subtitle:
---

## 서론

이전에 [중복 요청을 방지하기 위한 간단한 분산 락을 구현하는 글](https://swimminghwang.github.io/redis/2024/10/24/distributed-lock.html)을 작성했습니다.    
그 당시에는 어드민에서 사용하는 기능이라 레디스에 부하가 크게 걸리지 않을 뿐더러 높은 성능이 필요없었습니다. 하지만, 레디스에 부하를 줄 정도로 많은 요청이 있는 서비스의 경우 어떻게 구현을 하는지 궁금해졌었습니다.  
이번 글에서는 이러한 문제를 해결하기 위해, Redis 클라이언트 중 Jedis와 Lettuce와 같은 Redisson을 활용하여 더 빠르고 효율적으로 분산 락을 구현하는 방법을 공유하고자 합니다. 

### 들어가기 전, 

- Spring Data Redis : 레디스에 저장된 데이터를 조회하고 관리할 수 있게 해 주는 인터페이스를 제공하는 프레임워크  
- Spring Data Redis의 기본 클라이언트는 Lettuce

### 1. Lettuce

**Lettuce**는 **비동기, 동기, 리액티브 방식**을 모두 지원하는 고성능 Redis 클라이언트입니다. Lettuce는 주로 **고성능을 요구하는 멀티스레드 환경**에 적합합니다. 주요 특징은 다음과 같습니다:

- **비동기 및 리액티브 지원**: 넌블로킹 I/O와 비동기 작업을 지원해, 대규모 요청에서도 효율적인 성능을 발휘합니다.
- **스레드 세이프 구조**: 여러 스레드에서 동시에 접근하는 멀티스레드 환경에서도 안정적으로 동작합니다.
- **Spring Data Redis의 기본 클라이언트**: Spring Data Redis를 사용할 때 기본적으로 Lettuce를 클라이언트로 설정할 수 있습니다.
- **간단한 분산 락**: 기본적인 Redis 기반 분산 락 구현이 가능하지만, 분산 락이나 동기화 메커니즘이 Redisson에 비해 제한적입니다.

> Lettuce는 Redis에 대한 고성능 접근을 지원하고, **비동기와 리액티브 프로그래밍**을 선호할 때 적합한 선택입니다.

### 2. Redisson

**Redisson**은 Redis 클라이언트이면서도 **분산 시스템을 위한 고급 기능**을 제공합니다. Redisson은 특히 Redis를 **동기화와 분산 도구로 사용**할 때 유리하며, 이를 통해 복잡한 분산 아키텍처에 쉽게 Redis를 통합할 수 있습니다. 주요 특징은 다음과 같습니다:

- **Distributed Lock, Semaphore, Countdown Latch**와 같은 고급 분산 도구 지원: 동기화 메커니즘이 내장되어 있어, Redis를 단순한 데이터 스토리지 이상의 역할로 사용할 수 있습니다.
	- 분산 락: 여러 서버나 스레드가 공유 자원에 대한 접근을 조율할 때 사용되는 락.
	- 세마포어: 여러 개의 스레드가 접근할 수 있는 리소스의 개수를 제한하여 동시성을 제어하는 기법.
	- 카운트다운 래치: 일정한 수의 작업이 완료될 때까지 대기하게 하는 동기화 메커니즘.
- **분산 컬렉션 지원**: RMap, RList, RSet 등 다양한 분산 컬렉션을 제공하여, 데이터 처리를 단순화합니다.
- **Spring Data Redis와 통합 가능**: Spring 환경에서도 쉽게 사용할 수 있도록 Redisson과 Spring의 통합이 잘 되어 있습니다.
- **Fair Lock, 자동 해제**: 고급 분산 락을 제공하며, Fair Lock 자동 해제 옵션이 있어 안정성이 높습니다.

> Redisson은 Redis를 활용한 고성능 동기화 메커니즘과 분산 환경을 구축할 때 탁월하며, **분산 시스템 개발**이 주목적인 경우 매우 유용합니다.

### 분산 컬렉션

**Redis를 기반으로 동작**하지만, 일반적인 Redis 데이터 구조보다 **분산 환경에서 활용하기 위한 기능들이 추가된 컬렉션**입니다. 여기서 **분산**이라고 하는 이유는 **여러 서버나 애플리케이션 인스턴스에서 동기화된 상태로 데이터를 접근하고 사용할 수 있도록 설계**되었습니다.

Redisson 분산 컬렉션과 기본 Redis 데이터 구조의 차이점으로는

- **동시성 제어 및 동기화 기능**  
    Redisson 분산 컬렉션은 **내장된 락과 동기화 메커니즘**을 사용해 여러 클라이언트나 스레드가 동시에 데이터를 읽고 쓸 때의 문제를 해결합니다. 예를 들어 `RMap`에서 데이터에 접근할 때, 내부적으로 락을 걸어 데이터를 변경하는 동안 다른 클라이언트가 접근하지 않도록 제어할 수 있습니다.
    
- **일관성 유지**  
    여러 인스턴스가 분산 환경에서 하나의 데이터 소스를 사용하더라도, Redisson의 분산 컬렉션은 데이터 일관성을 유지해 줍니다. 반면, 일반 Redis 데이터 구조는 단순히 데이터 저장소 역할만 하므로, 여러 클라이언트가 동시에 접근할 때 데이터 일관성을 보장하지 않습니다.
    
- **편리한 API 제공**  
    Redisson 분산 컬렉션은 Java 표준 컬렉션 API와 유사한 형태로 사용할 수 있어서, **Java 개발자가 쉽게 접근**할 수 있고, Java 컬렉션의 일부 동작을 Redis 기반으로 사용할 수 있습니다.
    
- **고급 기능**  
    일반 Redis 데이터 구조는 단순히 데이터를 추가, 수정, 삭제하는 수준이지만, Redisson의 분산 컬렉션은 **공정 락, 분산 락, 카운트다운 래치** 등 고급 기능을 결합하여, 분산 시스템의 요구사항에 맞게 안정적이고 신뢰성 있는 접근을 보장합니다.


### Lettuce와 다른 Redisson의 장점

- 스핀 락 형태가 아닌 pub/sub 형식을 이용하여 락을 획득하고 해제하여 레디스 부하를 줄일 수 있습니다. 
- Lock에 타임아웃을 지정할 수 있습니다.



다음으로는 간단한 실습 코드입니다.

### Redisson을 이용한 분산 락 구현

다음은 Redisson을 사용해 빠르고 효율적인 분산 락을 구현하는 방법입니다.

#### 1. 의존성 추가

```groovy
dependencies {
    implementation 'org.redisson:redisson-spring-boot-starter:3.19.0'
}
```


#### 2. Redisson을 이용한 분산 락 코드 예제

아래 코드는 Redisson을 사용하여 **동기 분산 락**을 구현하는 예제입니다.

```java
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class RedissonLockService {

    private final RedissonClient redissonClient;

    public RedissonLockService(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    public void performLockedOperation(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 10초 동안 락을 시도하고, 락이 성공하면 5초 후 자동 해제
            if (lock.tryLock(10, 5, TimeUnit.SECONDS)) {
                // 락이 걸린 상태에서 실행할 로직
                System.out.println("작업이 성공적으로 수행되었습니다.");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            logger.error("락 작업 중 인터럽트 발생: " + e.getMessage());
        } finally {
            lock.unlock();
        }
    }
}

```

- **`tryLock`** 메서드는 지정된 시간 동안 락을 시도하고, 락이 성공하면 5초 후 자동 해제됩니다.
- **자동 해제 기능**은 락이 오래 걸려 있는 경우에도 안전하게 시스템이 동작하도록 해줍니다.

### 결론
- **Lettuce**는 **단순한 데이터 접근이나 캐싱**을 수행할 때 빠릅니다.
- **Redisson**은 **고급 분산 락과 동기화 도구**가 필요할 때 안정적이고 효율적입니다.

복잡한 분산 환경에서 높은 안정성과 동기화가 필요한 경우 Redisson이 더 적합하며, 단순한 데이터 처리가 주 목적이라면 Lettuce가 좋은 선택이 될 수 있습니다.