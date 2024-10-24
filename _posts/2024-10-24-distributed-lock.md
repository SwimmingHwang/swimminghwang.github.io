---
layout: post
title: Redis와 AOP를 활용한 락 구현하기
author: Soo-young Hwang
tags:
  - redis
  - AOP
  - Spring-Boot
  - kotlin
category: redis
published: true
subtitle: 요청에 대한 중복 처리를 제한하기위해
---
## **서론**

프로젝트에서 발생한 문제는 요청이 짧은 순간에 중복으로 들어와 동일한 데이터가 중복으로 데이터베이스에 Insert 되는 것이었습니다.      
특히, 가격 데이터를 관리하는 시스템에서 어드민 사용자들이 가격을 업데이트할 때, 여러 명이 동시에 동일한 요청을 보내는 상황이 발생할 수 있었습니다.   
이로 인해 동일한 가격 정보가 중복으로 저장되는 문제와 데이터 무결성 문제가 발생할 가능성이 있었습니다.    
가격 데이터는 시스템에서 매우 중요한 정보였기 때문에, 중복 저장을 철저히 방지하고, 어드민 유저가 동시에 요청할 수 없도록 하는 강력한 해결책이 필요했습니다.    


## **고민한 해결 방법들**

중복 요청을 방지하기 위해 몇 가지 접근 방식을 고려했습니다

1. **프론트엔드에서의 중복 요청 방지**
    - 버튼 비활성화 등으로 사용자가 동일한 요청을 여러 번 보낼 수 없도록 하는 방법을 생각했습니다. 하지만 프론트엔드에서만 이를 막기에는 확실한 해결책이 되지 못하므로 백엔드에서 중복 요청을 방지하는 방법이 더 필요하다고 판단했습니다.
2. **데이터베이스를 통한 중복 요청 처리**
    - 각 요청에 대해 DB에 INSERT하고 데이터중복 여부를 체크하는 방법도 고려했습니다. 그러나 이 방법은 데이터베이스에 부하를 줄 수 있고, 간단한 체크 작업에 비해 데이터베이스 요청이 과도할 수 있었습니다.
3. **Redis를 활용한 중복 요청 방지**
    - Redis의 캐싱 기능을 이용하여, Redis에 저장하고, 처리하는 동안 요청이 발생하면 처리되지 않도록 하는 방식을 선택했습니다. Redis는 메모리 기반의 저장소이므로 매우 빠른 조회와 저장이 가능하고, 분산 환경에도 적합했습니다.

## **선택한 해결 방법**

Redis를 활용하여 **중복 요청 문제**를 효과적으로 해결했습니다. 분산 환경에서 **동일한 요청**이 중복 처리되는 것을 방지하기 위해, 오직 하나의 요청만 처리되도록 **Redis를 이용한 락 메커니즘**을 구현했습니다.

이를 위해, **컨트롤러 단에서 AOP(Aspect-Oriented Programming)**를 적용한 **어노테이션**을 작성하고, 함수 실행 전에 해당 함수명을 기반으로 **고유한 키**를 Redis에 저장하여 **락을 설정**했습니다. **함수가 완료된 후에만 락이 해제**되어, 이후 요청들이 처리될 수 있도록 구현하여, 요청 처리의 **일관성과 안정성**을 확보했습니다. 이렇게 함으로써 해당 컨트롤러에서의 처리 과정 중 **동일한 요청**이 **중복으로 실행**되지 않도록 했습니다.


## 흐름도
![https://swimminghwang.github.io/assets/images/Pastedimage20241024223138.png](https://swimminghwang.github.io/assets/images/Pastedimage20241024223138.png)

## 작성한 코드 
### 1. `RedisLock` 어노테이션

```kotlin
@Target(AnnotationTarget.FUNCTION)  
@Retention(AnnotationRetention.RUNTIME)  
annotation class RedisLock (  
    val expiration: Long = 10,  
)
```
- **`@Target(AnnotationTarget.FUNCTION)`**: 이 어노테이션은 **함수**에만 적용될 수 있습니다.
- **`@Retention(AnnotationRetention.RUNTIME)`**: 이 어노테이션은 **런타임**까지 유지되어, 실행 중에 리플렉션을 통해 참조될 수 있습니다.
- **`expiration`**: 이 속성은 **락의 만료 시간**을 설정하는 데 사용됩니다. 기본값은 10초이며, 락이 걸린 후 만료되기까지의 시간을 나타냅니다.


### 2. `RedisLockAspect` 클래스 

```kotlin
@Aspect
@Component
class RedisLockAspect(
    private val redisRequestLockRepository: RedisRequestLockRepository
) {
    @Around("@annotation(redisLock)")
    @Throws(Throwable::class)
    fun lock(joinPoint: ProceedingJoinPoint, redisLock: RedisLock): Any? {
        val isLocked = redisRequestLockRepository.getLock(joinPoint.signature.name)
        return if (!isLocked) {
            try {
                joinPoint.proceed()
            } finally {
                redisRequestLockRepository.deleteLock(joinPoint.signature.name)
            }
        } else {
            throw FailException(StatusType.BLOCK_DUPLICATION_REQUEST)
        }
    }
}
```

- **`@Aspect`**: 이 클래스는 **AOP(Aspect-Oriented Programming)**에서 사용하는 **Aspect**로 정의되어, 특정 시점에서 실행 흐름을 가로채어 부가적인 로직을 실행할 수 있습니다.
- **`@Component`**: 스프링 컨테이너에 **빈(bean)**으로 등록되어, 의존성 주입을 통해 사용됩니다.
- **`RedisRequestLockRepository`**: Redis를 통해 요청에 대한 **락(lock)**을 관리하는 저장소 역할을 합니다. 락을 설정하거나 해제하는 등의 작업을 담당합니다.
- **`@Around("@annotation(redisLock)")`**: 특정 어노테이션(`@RedisLock`)이 적용된 함수의 실행 전후를 가로채어 **락 관련 로직**을 처리합니다.
- **`joinPoint: ProceedingJoinPoint`**: 이 객체는 실제로 실행될 **대상 메서드**를 나타냅니다. 이 메서드에 대한 정보를 조회하고, 해당 메서드를 실행할 수 있게 해줍니다.
- **`isLocked`**: `redisRequestLockRepository`를 통해 현재 함수(메서드명)와 관련된 **락이 존재하는지** 확인합니다.
    - **락이 없는 경우**: `joinPoint.proceed()`를 호출해 메서드를 실행하고, 이후 `deleteLock()`을 통해 락을 해제합니다.
    - **락이 있는 경우**: 중복 요청을 방지하기 위해 `FailException`을 던져 **요청을 차단**합니다.

### 3.  컨트롤러 (redislock 어노테이션 적용한 부분)
```kotlin
@RedisLock(expiration = 15)  
@PostMapping(Routes.API_PRICE_TEMPLATE_SYNC_GOOGLE_SHEET)  
fun syncBetweenGoogleSheetAndDb(  
    @RequestParam("type") type: Int,  
): BaseResponse<List<PriceTenmplate>> {  
    return BaseResponse(  
        data = priceTemplateService.syncGoogleSheetAndDb(PriceTemplateType.ofCode(type))  
    )}
```
- **`@RedisLock(expiration = 15)`**: 이 **어노테이션**을 락 하고자 하는 함수에 적용하고, 적용된 **함수가 실행되는 동안** 동일한 **HTTP 요청**이 중복으로 들어올 경우 해당 요청을 **차단**하고 **중복 처리를 방지**합니다.

## **결과**

**어드민 유저가 동시에 동일한 가격 데이터를 업데이트하는 상황을 방지**할 수 있었고, 이로 인해 시스템의 안정성과 일관성이 크게 향상되었습니다.