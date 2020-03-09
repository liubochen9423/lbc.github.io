```java
@Component
public class DefaultInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("interceptor prehandler");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {
        System.out.println("interceptor posthandler");
    }

}

@Configuration
public class DefaultInterceptorConf extends WebMvcConfigurationSupport {

    @Autowired
    private DefaultInterceptor interceptor;

    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        // 多个拦截器组成一个拦截器链
        // addPathPatterns 用于添加拦截规则，/**表示拦截所有请求
        // excludePathPatterns 用户排除拦截
        registry.addInterceptor(interceptor).addPathPatterns("/**")
                .excludePathPatterns("/stuInfo/getAllStuInfoA", "/account/register");
        super.addInterceptors(registry);
    }
}

```


```java
@Component
public class DefaultFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) {
        System.out.println("========= init doFilter ===========");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("========= before doFilter ===========");
        chain.doFilter(request, response);
        System.out.println("========= after doFilter ===========");
    }

    @Override
    public void destroy() {
        System.out.println("========= destroy doFilter ===========");
    }

}
```

```java
@Component
@Aspect
public class DefaultAspect {

    @Pointcut("execution(public * com.youzan.test.tata.biz.impl..*.*(..))")
    public void pointcut() {
    }

    @Around("pointcut()")
    public Object function(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        initMDC();
        System.out.println("========= before aspect ===========");
        Object proceed = proceedingJoinPoint.proceed();
        System.out.println("========= after aspect ===========");
        MDC.clear();
        return proceed;
    }

    private void initMDC() {
        MDC.put("INNER_TRACE_ID", generate());
    }

    public static String generate() {
        long currentTime = System.currentTimeMillis();
        long timeStamp = currentTime % 1000000L;
        int randomNumber = ThreadLocalRandom.current().nextInt(10000);
        long traceID = timeStamp * 10000 + randomNumber;
        return Long.toHexString(traceID).toUpperCase();
    }
}
```