基础设计
====
example:
令牌生成速率为 1/s,10秒内没有请求进来，此时拥有的令牌总数为10*1=10个（假定可容纳总令牌数>=10）；此时请求进来，需求令牌数=3，此时拥有的令牌总数为10-3=7个；请求再次进来，需求令牌数=10，消耗原有的7个，并且生成新的10-7个令牌。
所以生成这3个新的令牌需要多久？3*1/s=3s。但是我们并不知道消耗7个令牌所需要的总时间。策略1:关注于未使用资源，此时生成的新令牌要实时的放入拥有的令牌总数中。策略2:关注于对溢出资源的处理（把多出来的3个令牌预占），此时需要将拥有的令牌数转化为限流时间（后续限流x秒不可生成新的令牌）


```java
    /**
     *构造方法：这里使用了maxBurstSeconds 即单个令牌最大持有时间
     */
    SmoothBursty(SleepingStopwatch stopwatch, double maxBurstSeconds) {
      super(stopwatch);
      this.maxBurstSeconds = maxBurstSeconds;
    }
```

```java
  /**
   * The currently stored permits.当前令牌总数
   */
  double storedPermits;

  /**
   * The maximum number of stored permits.最大令牌数量 = 令牌生成速度 * 令牌持有时间
   */
  double maxPermits;

  /**
   * The interval between two unit requests, at our stable rate. E.g., a stable rate of 5 permits
   * per second has a stable interval of 200ms.
   * 令牌生成的时间间隔 5个/1s = 200ms间隔
   */
  double stableIntervalMicros;

  /**
   * The time when the next request (no matter its size) will be granted. After granting a
   * request, this is pushed further in the future. Large requests push this further than small
   * requests.
   *下一个请求被允许的时间点 example中说的后续限流时间，可能past or future，past就是正常操作，future即预消费
   */
  private long nextFreeTicketMicros = 0L; // could be either in the past or future
```

```java
    //实例化
    public static RateLimiter create(double permitsPerSecond) {
        return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
    }

    static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
        RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
        rateLimiter.setRate(permitsPerSecond);
        return rateLimiter;
    }
```

```java
    public final void setRate(double permitsPerSecond) {
    checkArgument(
        //permits-per-second为create(permitsPerSecond)传入参数 即 每秒生成的令牌数
        permitsPerSecond > 0.0 && !Double.isNaN(permitsPerSecond), "rate must be positive");
    synchronized (mutex()) {            //保证线程安全
        doSetRate(permitsPerSecond, stopwatch.readMicros());
    }
    }

    @Override
    final void doSetRate(double permitsPerSecond, long nowMicros) {
    resync(nowMicros); //重制可以生成令牌的时间，修改了生成令牌速度，必定要重新计算桶可持有令牌数量
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond; //生成令牌间隔 = 1s / 每秒生成令牌数量
    this.stableIntervalMicros = stableIntervalMicros;
    doSetRate(permitsPerSecond, stableIntervalMicros);
    }

    @Override
    void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
        double oldMaxPermits = this.maxPermits;
        maxPermits = maxBurstSeconds * permitsPerSecond;    //最大数量 = 最大持有时间 * 生成令牌速度
        if (oldMaxPermits == Double.POSITIVE_INFINITY) {
        // if we don't special-case this, we would get storedPermits == NaN, below
        storedPermits = maxPermits;
        } else {
        storedPermits = (oldMaxPermits == 0.0)
            ? 0.0 // initial state
            : storedPermits * maxPermits / oldMaxPermits; // 按比例放大当前存储的令牌数量
        }
    }
```


```java
  private void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now 
    if (nowMicros > nextFreeTicketMicros) {
    //若当前时间>下次令牌生成时间 则证明这期间有令牌生成，当前令牌数 = 老令牌数+（中间时间）/生成间隔 其实直接*速度向下取整也行 <= 最大可持有令牌数 。重定向下次生成时间为now。一句话总结：获取令牌时生成令牌 可生成令牌的时间点为nextFreeTicketMicros（很关键）
      storedPermits = min(maxPermits,
          storedPermits + (nowMicros - nextFreeTicketMicros) / stableIntervalMicros);
      nextFreeTicketMicros = nowMicros;
    }
  }
```

```java
  /**
   * Reserves next ticket and returns the wait time that the caller must wait for.
   *
   * @return the required wait time, never negative
   */
  final long reserveAndGetWaitLength(int permits, long nowMicros) {
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    return max(momentAvailable - nowMicros, 0);
  }

      /**
   * Reserves the requested number of permits and returns the time that those permits can be used
   * (with one caveat).获取得到需要令牌数的最短时间
     *
   * @return the time that the permits may be used, or, if the permits may be used immediately, an
   *     arbitrary past or present time
     */
  @Override
  final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;  //默认就是计算过的下次free令牌生成时间
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);  //判断是否要新增令牌 获取可消耗的存储的令牌数量； 供不应求：则最多消耗当前存储总值；供大于求：则消耗需求数量
    double freshPermits = requiredPermits - storedPermitsToSpend; //这个值>=0 ;供不应求的场景下 这个值为正 ,得到的结果值为需要生成的令牌数量

    long waitMicros = 0L
        + (long) (freshPermits * stableIntervalMicros);//对于平滑limiter是0+新令牌数*生成时间

    //这里重制下次令牌生成时间，和当前存储的所有令牌；这里的时间+waitMicros控制了预消费
    this.nextFreeTicketMicros = nextFreeTicketMicros + waitMicros;
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
  }
```

```java
  //申请令牌的调用，即调用reserveEarliestAvailable 并返回需要等待的时间
  public double acquire(int permits) {
    long microsToWait = reserve(permits);
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return 1.0 * microsToWait / SECONDS.toMicros(1L);
  }

  final long reserve(int permits) {
    checkPermits(permits);
    synchronized (mutex()) {
      return reserveAndGetWaitLength(permits, stopwatch.readMicros());
    }
  }

  final long reserveAndGetWaitLength(int permits, long nowMicros) {
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    return max(momentAvailable - nowMicros, 0);
  }
```