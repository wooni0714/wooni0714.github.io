---
layout: post
title: DistributedLock을 사용한 동시성 제어 해결
subtitle: 재고감소 및 한정수량 판매 시스템같은 양한 동시성 제어 문제를 맞닥뜨리면서 Redisson DistributedLock(분산락)과 AOP을 활용하여 이와 같은 문제를 해결
author: wooni
categories: spring
# tags: blog jekyll
image: ![image](https://github.com/user-attachments/assets/5941c0f1-0630-448e-bca3-d611f8512257)
featured: true
date: 2025-05-20
hidden: false
comments: true
permalink: '/:categories/:title'
---

## 1. 개요
- 재고감소 및 한정수량 판매 시스템같은 양한 동시성 제어 문제를 맞닥뜨리면서  Redisson DistributedLock(분산락)과 AOP을 활용하여 이와 같은 문제를 해결하고 이해하기 위해 다루었습니다.

프로젝트 환경 및 사용기술

- JAVA, SpringBoot, JPA, Redis, H2, Mysql
- Redisson(Distributed Lock), AOP

## 2. Redis의 Redisson을 선택한 이유

- 락을 구현하는 방식에는 JVM 내부 락, MySQL DB 락, Zookeeper 기반 락, Redis 기반 분산락 등이 있으며 현재 프로젝트에서는 MySQL 기반 락 대신 Redis의 Redisson 분산락을 선택했습니다.
- MySQL의 경우, 서버 간 락 상태를 공유할 수 없고, 락 경합이 발생하면 DB 부하가 집중되어 성능 저하로 이어질 수 있는 단점이 있습니다.
- 반면 Redis 기반의 Redisson은 분산 환경에서도 안정적으로 동시성 제어가 가능하며 락 해제 실패나 데드락 상황에서도 자동화된 복구를 통해 안정성을 보장할 수 있습니다.
- 또한 TTL, Watchdog 기능과 함께 AOP 기반 선언적 락 처리도 지원되어 복잡한 코드 수정 없이 Race Condition을 효과적으로 방지할 수 있습니다.

## 3. Redisson과 Lettuce

### Lettuce

- Lettuce는 Redis 클라이언트로서, 분산락 기능이 내장되어 있지 않기 때문에 setnx, setex 등을 활용해 직접 락 구현을 해야 하며, 락 획득을 위해 Redis에 지속적으로 요청을 보내는 Spin Lock 방식을 사용하게 됩니다. 동시에 많은 스레드가 락을 기다릴 경우 Redis에 과도한 부하가 일어납니다.
    <p align="center">
  <img src="https://github.com/user-attachments/assets/e3850f8c-2521-4b2d-a9f9-49e33a2c97d4" width="500" height="500" />
  </p>
    1. Thread-1의 키가 1인 데이터를 redis에 set하면 맨 처음엔 1이 없으므로 성공.
    2. Thread-2가 키가 1인 데이터를 redis에 set하면 1이 존재함으로 실패.
    3. Thread-2는 성공을 하기 위해 100초마다 재시도 하는 로직을 만들어 성공할 때 까지 함.
 
  ### Redisson

- **Redisson은** pub-sub 방식으로  락이 해제되면 락을 subscribe 하고 있는 클라이언트에게 락이 해제되었다는 알림을 보내 락 획득을 시도하기에 Lettuce와 비교했을 때 Redis에 부하가 덜 갑니다.
- `Lock interface` 지원하기 때문에 waitTime, leaseTime 등의 타임아웃 설정이 가능하고 내부적으로 Watchdog 기능도 제공하여 더 안정적이고 안전한 락 제어가 가능합니다.
    
    ![Image](https://github.com/user-attachments/assets/c99c123e-3000-409f-af29-b2115d52ed2f)
  
    1.  채널을 구독함으로 Thread-1의 Lock 사용종료를 채널에 알림
    2.  Thread-2는 구독한 채널을 통해 Thread-1의 Lock 점유가 해제됨을 알고 Lock을 획득

## 4. AOP (Aspect-Oriented-Programming)

- AOP는 로깅, 트랜잭션 등 처럼 반복적으로 사용되는 공통 기능을 핵심 비즈니스 로직과 분리하여 모듈화 할 수 있는 프로그래밍 기법 입니다.
- 코드상에서 계속 반복해서 사용되는 부분들을 흩어진 관심사(Crosscutting Concerns)라고 하고 Aspect 형태로 분리합니다.
- **Aspect는** 공통 기능을 담은 모듈로 어디에 어떻게 적용할지를 `Pointcut`과 `Advice`를 통해 정의합니다.
- 객체지향적으로 코드를 짤 수 있게 도와주며 유지보수가 용이해집니다.

### AOP 동작과정

<img width="1044" alt="Image" src="https://github.com/user-attachments/assets/30110fd9-22be-41f1-a558-eaacab15ee2f" />

## 5. Redisson 적용

1. DistributedLock 분산락 적용 다이어그램
<p align="center">
  <img src="https://github.com/user-attachments/assets/2adf42d1-3c13-483c-a693-ba98e85f507a" width="500" height="800" />
</p>
흐름

- 구매 요청이 들어오면, 분산락 어노테이션을 통해 해당 상품에 대한 락을 시도합니다.
- 락 획득 성공 : 별도의 트랜잭션(@Transactional(REQUIRES_NEW)) 안에서 재고 수량을 검증하고 차감하는 로직이 실행됩니다.
- 재고 차감이 성공적으로 완료되면, 트랜잭션이 커밋되고 이후 락이 해제됩니다.
- 락 획득 실패 : 정해진 횟수만큼 일정 시간 간격으로 락 재시도를 수행합니다.
- 재시도 중에도 락 획득에 계속 실패하거나 재고 부족 등 비즈니스 예외가 발생하면 구매는 실패로 처리됩니다.

AOP

- AOP를 활용해 락의 획득과 해제 로직을 분리함으로써, 중복 코드를 제거하고 보일러플레이트 문제를 해결할 수 있습니다.
- `@Around`,`@DistributedLock` 어노테이션을 사용해 락이 필요한 메서드를 선언적으로 지정하여 사용합니다.

build.gradle

```java
dependencies {
	//...
    implementation 'org.redisson:redisson-spring-boot-starter:3.17.0'
}
```

RedissonConfig

```java
@Configuration
public class RedissonConfig {

    @Value("${spring.data.redis.host}")
    private String redisHost;

    @Value("${spring.data.redis.port}")
    private int redisPort;

    private static final String REDISSON_HOST_PREFIX = "redis://";

    @Bean
    public RedissonClient redissonClient() {
        RedissonClient redisson = null;
        Config config = new Config();
        config.useSingleServer().setAddress(REDISSON_HOST_PREFIX + redisHost + ":" + redisPort);
        redisson = Redisson.create(config);
        return redisson;
    }
}

```

DistributeLock

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {

    String key(); // 락 이름

    long waitTime() default 5L; // 락을 획득하기 위한 대기시간

    long leaseTime() default 3L; // 락 획득 후 자동 해제시간

    TimeUnit unit() default TimeUnit.SECONDS; // waitTime, leaseTime 시간단위
}
```

- waitTime은 5~10초사이로 설정하고 락 획득이 실패하면 빠르게 획득 실패처리를  하고 leaseTime은 트랜잭션 시간보다 여유있게 설정되어야 합니다. 예를 들어 트랜잭션이 5초안에 끝난다면 leaseTime은 10초이상으로 설정하여 트랜잭션 도중 락이 해제되는 문제를 방지해야 합니다.

DistributedLockAspect

```java
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class DistributedLockAspect {
    private final RedissonClient redissonClient;
    private final AopTransactionAspect aopTransactionAspect;
    private static final String REDISSON_LOCK_PREFIX = "LOCK:";

    @Around("@annotation(com.limited.product.common.aop.DistributedLock)")
    public Object lock(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        DistributedLock distributedLock = method.getAnnotation(DistributedLock.class);

        String key = createLock(distributedLock, methodSignature, joinPoint);
        RLock rLock = redissonClient.getLock(key);

        try {
            boolean isTriedLock = rLock.tryLock(distributedLock.waitTime(), distributedLock.leaseTime(), distributedLock.unit());
            if (!isTriedLock) {
                throw new IllegalStateException("Lock 획득 실패");
            }

            return aopTransactionAspect.proceed(joinPoint);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new InterruptedException();
        } finally {
            if (rLock.isHeldByCurrentThread()) {
                try {
                    rLock.unlock();
                } catch (IllegalStateException e) {
                    log.warn("Unlock failed name = {}, key = {}", method.getName(), key);
                }
            }
        }
    }

    private String createLock(DistributedLock distributedLock, MethodSignature methodSignature, JoinPoint joinPoint) {
        return REDISSON_LOCK_PREFIX + SpELExpressionEvaluator.evaluate(
                methodSignature.getParameterNames(),
                joinPoint.getArgs(),
                distributedLock.key());
    }
}

```

SpELExpressionEvaluator

```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class SpELExpressionEvaluator {

    public static Object evaluate(String[] parameterNames, Object[] args, String key) {
        ExpressionParser parser = new SpelExpressionParser();
        StandardEvaluationContext context = new StandardEvaluationContext();

        for (int i = 0; i < parameterNames.length; i++) {
            context.setVariable(parameterNames[i], args[i]);
        }

        return parser.parseExpression(key).getValue(context, Object.class);
    }
}
```

AopTransactionAspect

```java
@Component
public class AopTransactionAspect {
    @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 2)
    public Object proceed(final ProceedingJoinPoint joinPoint) throws Throwable {
        return joinPoint.proceed();
    }
}
```

Entity

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Slf4j
@Table(name = "LIMITED_PRODUCT")
public class LimitedProduct {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long productId;

    private Long quantity;

    private int maxQuantity;

    @Builder
    public LimitedProduct(Long productId, Long quantity, int maxQuantity) {
        this.productId = productId;
        this.quantity = quantity;
        this.maxQuantity = maxQuantity;
    }

    public void validateMaxQuantity(Long requestedQuantity) {
        if (requestedQuantity > this.maxQuantity) {
            throw new BusinessException(EXCEEDS_MAX_PURCHASE_QUANTITY + this.maxQuantity);
        }
        if (this.quantity - requestedQuantity < 0) {
            throw new BusinessException(INSUFFICIENT_STOCK);
        }
    }

    public void decreaseQuantity(Long quantity) {
        this.quantity -= quantity;
    }
}
```

Service

```java
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class LimitedProductStockService {
    private final LimitedProductRepository limitedProductRepository;

    @DistributedLock(key = "#productId")
    public void decreaseStock(Long productId, Long quantity) {
        LimitedProduct product = limitedProductRepository.findByProductId(productId)
                .orElseThrow(() -> new BusinessException(PRODUCT_NOT_FOUND));

        try {
            product.validateMaxQuantity(quantity);
            product.decreaseQuantity(quantity);
        } catch (BusinessException e) {
            log.error("[구매 실패] productId={}, userQuantity={}, stock={}, maxPerUser={}", productId, quantity, product.getQuantity(), product.getMaxQuantity());
            throw e;
        }

        limitedProductRepository.save(product);
    }
}
```

TEST

```java
@SpringBootTest
class LimitedProductServiceTest {

    @Autowired
    private LimitedProductStockService productService;

    @Autowired
    private LimitedProductRepository productRepository;
    
    @Autowired
    private RedissonClient redissonClient;

    @BeforeEach
    void setUp() {
        productRepository.saveAndFlush(new LimitedProduct(1L, 100L, 1));
        memberRepository.flush();
    }

    @AfterEach
    void after() {
        productRepository.deleteAll();
        redissonClient.getKeys().deleteByPattern("LOCK:*");
    }
    
    @Test
    void 동시에_100개_요청() throws InterruptedException {
        int threadCount = 100;
        ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
        CountDownLatch countDownLatch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            executorService.submit(() -> {
                try {
                    productService.decreaseStock(1L, 1L);
                } finally {
                    countDownLatch.countDown();
                }
            });
        }
        countDownLatch.await();

        LimitedProduct product = productRepository.findById(1L).orElseThrow();

        assertEquals(0, product.getQuantity());
    }
    
    @Test
    void 분산락_미적용_동시에_100개_요청() throws InterruptedException {
        int threadCount = 100;
        ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
        CountDownLatch countDownLatch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            executorService.submit(() -> {
                try {
                    productService.decreaseStock(1L, 1L);
                } finally {
                    countDownLatch.countDown();
                }
            });
        }
        countDownLatch.await();

        LimitedProduct product = productRepository.findById(1L).orElseThrow();
        log.info("구매 수량: {}", product.getQuantity());
        assertEquals(0, product.getQuantity());
    }
}
```

DistributedLock 적용 전

![Image](https://github.com/user-attachments/assets/d6514a2a-4cf6-4d96-87e6-b2bae6b79cf2)

JMeter 테스트

![Image](https://github.com/user-attachments/assets/68d2c619-9b1f-4e73-8703-6874eff98cbe)

- 수량 100개 한정 상품 등록
<p align="center">
  <img src="https://github.com/user-attachments/assets/30f06c07-fc66-4129-bf9f-ae34c9d4cbaf" width="560" height="600" />
  <img src="https://github.com/user-attachments/assets/804e0ce2-e672-4de9-a020-b36ea494b3d5" width="350" height="200" />
</p>

- DistributedLock 분산락을 적용하기 전에는 재고수량 100개에 대해 동시에 110개의 쓰레드 요청에 모두 성공처리 되었지만 실제로는 69개의 수량이 남아 중복 처리로 인한 동시성 문제가 발생했습니다.

DistributedLock 적용 후

<img src="https://github.com/user-attachments/assets/1045aee8-9967-4cd9-8120-efdb954f7f8c" width="350" height="100" />
<br />

<p align="center">
  <img src="https://github.com/user-attachments/assets/740a4181-87a4-4e9f-9408-54f447d7da41" width="560" height="600" />
  <img src="https://github.com/user-attachments/assets/e3f37d09-7644-498c-a0be-dfe003cd90d7" width="350" height="200" />
</p>

- DistributedLock 분산락을 적용한 후에는 재고수량 100개에 대해 110개의 쓰레드를 동시에 요청을 보낸 결과 100개만 락을 획득해 성공하였고 나머지 10개는 수량부족으로 락 획득을 실패하여 처리되지 않았으며 실제 재고 수량도 0개로 정확히 소진되어 동시성제어가 정상적으로 이루어졌습니다.

## **6. Issue**

### Transaction commit 이후 락이 해제되어야 하는 이유

1. 락의 해제 시점이 트랜잭션 커밋 이전일 경우

<p align="center">
  <img src="https://github.com/user-attachments/assets/1c52eb1c-b7ef-4904-95fc-70610ae1535a" width="560" height="700" />
</p>

1. Client 1과 Client 2가 동시에 요청시 Client 1이 먼저 락을 획득하고 재고를 조회합니다. (재고 조회:100개)
2. Client 1은 재고를 1개 차감한 후 트랜잭션 커밋 이전에 락을 먼저 해제합니다.(재고 99개)
3. Client 2는 락이 해제됨을 감지하고 락을 획득하여 재고를 조회 합니다.(재고 조회:100개)
4. Client 2는 재고 1개를 차감한 후 락을 해제하고 커밋합니다.

— 두 Client 중 하나의 커밋이 다른 하나를 덮어씌우게 되어 실제로는 1개만 차감된 것처럼 정합성 문제가 발생합니다.

2. 락의 해제 시점이 트랜잭션 커밋 이후일 경우

<p align="center">
  <img src="https://github.com/user-attachments/assets/9bc97331-4ce4-48df-9ce5-169d26a2c69f" width="560" height="700" />
</p>

1. Client 1과 Client 2가 동시에 요청시 Client 1이 먼저 락을 획득하고 재고를 조회합니다. (재고 조회:100개)
2. Client 1은 재고를 차감한 후 트랜잭션 커밋을 하고 락을 해제합니다. (재고:99개)
3. Client 2는 락이 해제되었다는 알림을 받고 락을 획득하여 재고를 조회 합니다.(재고 조회:99개)
4. Client 2는 재고를 차감한 후  트랜잭션 커밋을 한 뒤 락을 해제 합니다. (재고:98개)

— Client 1 과 Client 2는 락의 해제가 트랜잭션 커밋 이후에 이루어지기 때문에 직전 트랜잭션이 반영된 재고 값으로 데이터의 정합성이 보장됩니다.

### 데이터 베이스  커넥션 풀 부족으로 인한 데드락

AopTransactionAspect에서 `@Transactional(propagation = Propagation.REQUIRES_NEW`)는 새로운 트랜잭션을 생성하는데 DB커넥션이 또한 새로 생성됩니다. Thread 간에 커넥션을 선점하기 위한 Race Condition이 발생한 상태에서 커넥션 풀이 고갈되면 일부 쓰레드는 커넥션을 획득하지 못하고 대기하거나 트랜잭션 타임아웃 또는 데드락이 발생합니다.

동시에 다수의 요청이 `REQUIRES_NEW`를 사용하는 경우 커넥션 풀의 설정값보다 많은 트랜잭션이 생성되어 커넥션이 블로킹되고 전체 응답 지연과 시스템 병목의 원인이 됩니다.

해결방법

1. 커넥션 풀 크기 설정

```java
spring.datasource.hikari.maximum-pool-size: 101
```

<aside>
💡

데드락을 피할수 있는 최소한의 pool size 공식

poo size = Tn x (Cm -1) + 1

Tn(thread count) : 100

Cm(하나의 요청에 동시에 필요한 커넥션 수) : 2

100 * (2 -1)  + 1 = 101

</aside>

- @Transactional(REQUIRES_NEW)를 사용하기 때문에 기존 트랜잭션과 별도의 DB 커넥션을 새로 할당 받으므로 최소 2개의 트랜잭션이 실행됩니다.

**References**

https://twosky.tistory.com/64
https://techblog.woowahan.com/2663/
https://techblog.woowahan.com/2664/
https://helloworld.kurly.com/blog/distributed-redisson-lock/#reference
