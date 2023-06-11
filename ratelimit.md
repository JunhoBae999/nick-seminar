# 가용성 확보를 위한 Rate Limit Algorithm
# Intro
##  Noisy Neighbor Problem

[image:6F1653A0-AC89-443C-8572-F9A1D98C18B7-616-000000093ACBF3E5/76388B62-6DB7-448B-964E-9B1CD02C3E30.png]

서비스의 client들 중에서 갑자기 어느 한 client가 무수히 많은 요청을 보내기 시작한다면?
* 클라이언트 역시 갑자기 트래픽이 몰리는 경우
* 부하 테스트를 하는 경우
* 악의적인 클라이언트가 ddos 공격을 하는 경우

즉, 하나의 클라이언트가 서비스 호스트의 cpu,memory, disk, network i/o를 과도하게 사용하려 하는 경우,
*다른 선량한 클라이언트는 높은 latency / request failure를 경험할 수 있음.*
이러한 Noisy Neighbor Problem을 방지하기 위한 여러 옵션 중 하나로 Rate Limit(Throttling)을 고려할 수 있음

# Rate Limit
## Rate Limit?
[image:A3341FC7-38B6-4952-A70F-00CA9225B7DC-616-000000098675095B/7A8ED6D9-EB18-4905-8634-DE3E33AE04F1.png]

“traffic rate를 제한”하는 것. 즉, client의 과도한 과도한 사용에 대해 서비스가 스스로를 보호할 수 있게 하는, 혼잡 제어 기법
* *클라이언트가 일정 시간동안 보낼 수 있는 요청의 개수를 제한함*
* *limit를 넘은 요청은 무시되거나 지연됨.*

## why?
1 과도한 트래픽으로부터 서비스를 보호
	1 ex) ddos 공격으로부터 서비스를 보호
2 Resource 사용에 대한 공정성과 합리성 유도.
	1 api 클라이언트 간에 api resource 사용의 공평성을 제공. ex) 소수의 heavy 클라이언트로 인해 다른 클라이언트가 피해를 입지 않아야 함
3 트래픽 비용이 서비스 예산을 넘는 것을 방지.
4 Rate에 대해 과금을 부과하는 Business Model로 활용.

## 의문점

*부하를 핸들링하기 위해서 소프트웨어를 만들어야 할까? 우리 호스트의 클러스터를 scale-out, auto scaling 하면 되는거 아닐까?*

scale-up, scale-out.. auto-scaling은 즉시 이루어지지 않음. 우리 서비스가 이미 crash되었을 수도 있음

*로드밸런서 쓰면 안되나?*
물론 로드밸런서가 트래픽 과부하를 방지할 수 있음. 하지만, 로드밸런서는 *무차별적임(indiscrminate)*

* 만약 어떤 서비스에서 여러 operation을 제공
* 여러 operation 중에서 어떤 operation은 비용이 많이 들고, 어떤 operation은 비용이 적게 듬
* 비용이 많이 드는 operation에 대해서만 request를 제한하고 싶다면, 어플리케이션에서 제한해야 함.

*하나의 호스트에 대해서만 rate-limit 기법을 적용하면 될까? 분산 시스템에서는 어떻게 될까?*

* 이상적으로는 그렇지만, real world에서는 그렇지 않음
* 로드밸런서가 있고, 로드밸런서가 여러 서버에 동일하게 요청을 전파하고, 모든 요청이 처리되는데 동일한 시간이 걸린다면? : single instance problem. 분산 시스템을 고려할 필요가 없음
* 하지만 real-world에서는
	1 로드밸런서가 완벽히 동일하게 요청을 전파하지 않음
	2 서비스의 operation 마다 cost가 다름
	3 각각의 서버마다 사양이 다를수도 있고, 상황이 다를수도 있음. 백그라운드 프로세스가 다를 수도 있고, 혹은 software failure로 느려졌을 수도 있음. 다양한 상황이 존재
* 따라서, 각각의 서버가 서로 지금까지 얼마나 많은 요청을 처리했는지 *communicate 할 수 있는 솔루션*이 필요함

# 요구사항
## Functional Requirements
1 allow(request)
	1 request를 제한할지 말지에 대한 boolean value를 return

## Non-Functional Requirement
1 Low Latency : rate limiting이 빨라야함! 모든 요청마다 호출되어야 하기 때문에
2 Accurate : 반드시 필요한게 아니라면 요청에 제한을 걸어서는 안됨
3 Scalable : 서비스의 scale-out과 호환되어야 함. (서비스 클러스터에 호스트를 늘리는데, rate-limiter가 문제가 되어서는 안됨)

*HA, Fault Tolerance에 영향을 미치지는 않을까?*
* Rate Limiting에서 어떤 failure로 결정을 하지 못한다면, *결론은 제한하지 않는 것*
* 모르겠다면 제한하지 않는다.

# Interface
```java 
 package com.mimul.ratelimit;
public abstract class RateLimiter {

 protected final int maxRequestPerSec;

 protected RateLimiter(int maxRequestPerSec) {
  this.maxRequestPerSec = maxRequestPerSec;
}

 abstract boolean allow();
}
```


# 간단한 시스템 Overview
[image:A6ED4273-E3E2-412F-BC21-565A5A9A418F-616-00000009D905E999/5381F82E-90A4-4A9E-9D04-1CF53C47A5F4.png]

1 Rules Retriever : 각각의 rule은 특정 클라이언트가 초당 얼만큼의 요청을 보낼 수 있는지 의미. 서비스 오너에 의해 정의되며, 데이터베이스에 저장됨. 얘는 주기적으로 룰을 가져오는 백그라운드 프로세스
2 리트리버가 캐시에 저장함
3 Client Identifier Builder : request가 오면, Client Identifier(key : ip, login…등등)를 만들어줌 (클라이언트 아이덴피케이션이 중요하다는걸 얘기하기 위해서… 굳이 분리한 것)
4 Rate Limiter : 키를 전달받아서 의사결정함. 캐시에서 룰을 가져와서 확인하고, 해당 클라이언트로부터의 요청을 처리할 수 있는지 확인.
5 거절된 경우, 특정 status code를 줄 수도 있고, 큐에 두고 나중에 처리할 수도 있음. 
* Hard Throttling : throttle limit을 엄격하게 관리 (1개라도 넘으면 안됨)
* Soft Throttling : 특정 percentage 까지는 throttle limit을 넘기는 것을 허용. ex) 분당 100회인데, 10%의 percentage를 둬서 110회까지는 허용
* Elastic or Dynamic Throttling : 서버 자원에 여유가 있을 때는 throttle limit을 넘기더라도 요청 허용

# 알고리즘
# Leaky Bucket 알고리즘
* 네트워크로부터의 데이터 주입 속도의 상한을 정해, 트래픽 체증을 일정하게 유지함
* 일정한 유출속도를 제한하여 유입 속도를 부드럽게함
* 고정 용량의 버킷에 다양한 유량의 물이 들어오면 버킷에 담기고, 물은 일정한 비율로 떨어짐
* 물의 양이 많아 넘치면 버림
* 즉, 입력 속도가 출력 속도보다 빠르면 누적이 발생하고, 누적이 버킷 용량보다 큰 경우 오버플로우가 발생, 패킷 손실 발생
* 여러 종류의 트래픽 속도를 지원해야 하는 경우에는 비 효과적, Peak rate가 고정된 값을 가지므로 네트워크 자원의 여유가 많을 때에도 충분히 활용할 수 있는 적응성을 가지지 못함.
* 채용 플랫폼 : Amazon MWS, NGINX, Shopify, Guava RateLimiter
```java
public class LeakyBucket extends RateLimiter {
  private final long capacity;
  private long used;
  private final long leakInterval;
  private long lastLeakTime;

  protected LeakyBucket(int maxRequestPerSec) {
    super(maxRequestPerSec);
    this.capacity = maxRequestPerSec;
    this.used = 0;
    this.leakInterval = 1000 / maxRequestPerSec;
    this.lastLeakTime = System.currentTimeMillis();
  }

  @Override
  boolean allow() {
    leak();
    synchronized (this) {
      this.used++;
      if (this.used >= this.capacity) {
        return false;
      }
      return true;
    }
  }

  private void leak() {
    final long now = System.currentTimeMillis();
    if (now > this.lastLeakTime) {
      long millisSinceLastLeak = now - this.lastLeakTime;
      long leaks = millisSinceLastLeak / this.leakInterval;
      if(leaks > 0) {
        if(this.used <= leaks){
          this.used = 0;
        } else {
          this.used -= (int) leaks;
        }
        this.lastLeakTime = now;
      }
    }
  }
}

```

# Token Bucket
* 일시적으로 많은 트래픽이 와도 토큰이 있다면 처리가 가능, 토큰 손실 처리를 통해 평균 처리 속도를 제한
* 토큰을 정해진 비율로 버킷에 넣음
* 버킷은 최대 N개의 토큰을 저장, 버킷이 가득 차면 새로 추가된 토큰은 삭제되거나 거부
* 요청이 들어올 때마다 토큰을 꺼내 부여하고, 요청이 처리된 후에는 토큰을 삭제
* 토큰이 배치되는 속도를 기반으로 액세스 속도를 제어
* 만약 토큰이 없다면 요청은 거절됨
* 토큰은 상수의 비율로 리필됨
* 간단하고 구현하기도 쉬움
* 채용 플랫폼 : Amazon MWS, NGINX, Shopify, Guava RateLimiter
```java
public class TokenBucket extends RateLimiter {
  private int tokens;
  private int capacity;
  private long lastRefillTime;

  public TokenBucket(int maxRequestPerSec) {
    super(maxRequestPerSec);
    this.tokens = maxRequestPerSec;
    this.capacity = maxRequestPerSec;
    this.lastRefillTime = scaledTime();
  }

  @Override
  public boolean allow() {
    synchronized (this) {
      refillTokens();
      if (this.tokens == 0) {
        return false;
      }
      this.tokens--;
      return true;
    }
  }

  private void refillTokens() {
    final long now = scaledTime();
    if (now > this.lastRefillTime) {
      final double elapsedTime = (now - this.lastRefillTime);
      int refill = (int) (elapsedTime * this.maxRequestPerSec);
      this.tokens = Math.min(this.tokens + refill, this.capacity);
      this.lastRefillTime = now;
    }
  }

  private long scaledTime() {
    return System.currentTimeMillis() / 1000;
  }
}

```

# Fixed Window Counter
* 정해진 시간 단위로 window가 만들어지고, 요청 건수가 기록되어 해당 window의 요청 건수가 정해진 건수보다 크면 해당 요청은 처리가 거부됨
* 경계의 시간대 (12:59, 13:01)에 요청이 오면 두배의 부하를 받게됨.
* 구현은 쉬우나 기간 경계의 트래픽 편향 문제 발생
* 버킷에는 시간 단위의 window 존재
	* window1 : 12:00 ~ 13:00
	* window2 : 13:00 ~ 14:00
* 분당 4건이 상한이라고 가정한다면, 9번 요청은 거부됨

```java

public class FixedWindowCounter extends RateLimiter {
  private final ConcurrentMap<Long, AtomicInteger> windows = new ConcurrentHashMap<>();
  private final int windowSizeInMs;

  protected FixedWindowCounter(int maxRequestPerSec, int windowSizeInMs) {
    super(maxRequestPerSec);
    this.windowSizeInMs = windowSizeInMs;
  }

  @Override
  boolean allow() {
    long windowKey = System.currentTimeMillis() / windowSizeInMs;
    windows.putIfAbsent(windowKey, new AtomicInteger(0));
    return windows.get(windowKey).incrementAndGet() <= maxRequestPerSec;
  }

  public String toString() {
    StringBuilder sb = new StringBuilder("");
    for(Map.Entry<Long, AtomicInteger> entry:  windows.entrySet()) {
      sb.append(entry.getKey());
      sb.append(" --> ");
      sb.append(entry.getValue());
      sb.append("\n");
    }
    return sb.toString();
  }
}


```


# Sliding Window Log
* Fixed Window Counter의 단점인 기간 경계 편향에 대응하기 위한 알고리즘
* 요청건의 로그를 관리해야 하기 때문에 구현과 메모리 비용이 높음
* 분당 2건의 요청 처리 가정, 12초와 24초 요청은 처리됨.
* 36초 요청은 거부됨
* 1분 25초의 요청이 들어오면, 12초,14초의 요청을 pop, 남아있는 요청은 하나이기 때문에 지금의 요청은 처리됨
```java
public class SlidingWindowLog extends RateLimiter {
  private final Queue<Long> windowLog = new LinkedList<>();

  protected SlidingWindowLog(int maxRequestPerSec) {
    super(maxRequestPerSec);
  }

  @Override
  boolean allow() {
    long now = System.currentTimeMillis();
    long boundary = now - 1000;
    synchronized (windowLog) {
      while (!windowLog.isEmpty() && windowLog.element() <= boundary) {
        windowLog.poll();
      }
      windowLog.add(now);
      log.info("current time={}, log size ={}", now, windowLog.size());
      return windowLog.size() <= maxRequestPerSec;
    }
  }

```

# Sliding Window Counter
* Fixed Window Counter의 경계 문제, Sliding Window Log의 로그 보관 비용 등의 문제점을 보완하는 알고리즘
* window의 위치를 바꿔봄
* rate을 계산하기 위해 timestamped log를 기록하면서, 이전 window의 request rate과 현재 window의 timestamp의 request rate에 weight를 먹여 계산하는 방식.
* 분당 10건 처리, 1분안에 9건의 요청, 1분부터 현재까지 5개의 요청이 왔다고 가정.
* 1분 15초에 요청이 왔는데, 이는
	* 1분과 2분 사이에서 25%지점
	* 이전 기간은 1-0.25 = 75% 비율로 계산
	* 9*0.75 + 5 = 14.75, 14.75 >10 이기 떄문에 한도 초과, 요청 거부
	* 즉 이전 window와 현재 window의 비율 값으로 계산된 합이 처리 건수를 초과하면 거부됨
* 이전 window에 대한 approximation을 하기 때문에 정확한 계산은 아니지만, 높은 정확도를 가지고 있음.
* 채용 플랫폼 : RateLimitJ

```java
public class SlidingWindow extends RateLimiter {
  private final ConcurrentMap<Long, AtomicInteger> windows = new ConcurrentHashMap<>();
  private final int windowSizeInMs;

  protected SlidingWindow(int maxRequestPerSec, int windowSizeInMs) {
    super(maxRequestPerSec);
    this.windowSizeInMs = windowSizeInMs;
  }

  @Override
  boolean allow() {
    long now = System.currentTimeMillis();
    long curWindowKey = now / windowSizeInMs;
    windows.putIfAbsent(curWindowKey, new AtomicInteger(0));
    long preWindowKey = curWindowKey - 1000;
    AtomicInteger preCount = windows.get(preWindowKey);
    if (preCount == null) {
      return windows.get(curWindowKey).incrementAndGet() <= maxRequestPerSec;
    }
    double preWeight = 1 - (now - curWindowKey) / 1000.0;
    long count = (long) (preCount.get() * preWeight + windows.get(curWindowKey).incrementAndGet());
    return count <= maxRequestPerSec;
  }
}

```


# Conclusion
# 서비스별 Rate Limit 정보
* Rate Limit을 적용하려면, RFC6585에서 429 Too Many Request HTTP 상태 코드를 사용하라고 제안
* Rate-Limit(허용되는 요청의 최대 수), RateLimit-Remainig(남은 요청 수), RateLimit-Reset(요청 최댓값이 재설정될때까지의 시간) 등을 Header에 같이 보내주면 좋다.
[image:404ED773-54C4-4CD0-9C38-B2074E26C9B0-616-0000000B4B4C3566/81B3CFED-92F5-4FF3-8D0C-66B45D4DB0BC.png]


# 적절한 알고리즘 선택
* 트래픽 패턴을 잘 분석하는 것이 선행되어야 함. 그리고 적절한 알고리즘을 선택해야 함.
* ex. 트래픽에 민감하지 않은 경우에는 Token Bucket, 그 외에는 Fixed Window, Sliding Window..
* 서비스 인프라 트래픽을 수용할 수 없는 시점에 도달했을 때 Rate Limit을 적용해야 하며, 외부 서비스에 영향을 최소화 하도록 노력 한 다음 Rate Limit을 적용해야 함
	* ex. Common API는 Rate Limit에 걸리지 않을 정도로 상한값을 높게 잡음
# -분산 환경에서의 Rate Limit-	
N/A
# 추가 자료
*  [라인 엔지니어링](https://engineering.linecorp.com/ko/blog/high-throughput-distributed-rate-limiter/) 
*  [Spring에 Bucket4j 적용하기](https://www.baeldung.com/spring-bucket4j) 
*  [Spring Cloud Gateway에서 RateLimit 적용하기](https://spring.io/blog/2021/04/05/api-rate-limiting-with-spring-cloud-gateway) 









