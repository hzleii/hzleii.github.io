---
title: '自定义Spring-boot-starter-security'
date: 2024-04-30
description: '自定义Spring-boot-starter-security流程'
# topic: leetcode
author: hzlei
banner: /assets/post/2024/spring-boot-starter-security/index.webp
cover: /assets/post/2024/spring-boot-starter-security/index.webp
article:
  type: tech # tech/story
poster: # 海报（可选，全图封面卡片）
  # topic: 标题上方的小字 # 标题上方的小字，可选
  headline: 自定义Spring-boot-starter-security流程。 # 必选
  # caption: 重点整理，包括一些框架特定的内容、特性，以及与其他框架的对比等。 # 标题下方的小字，可选
  color: hsl(93deg 48% 47%) # 标题颜色，可选，默认为跟随主题的动态颜色 # white,red...
---


## maven依赖

```xml
<!-- security启动器 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
    <version>3.3.2</version>
</dependency>
<!-- 依赖web环境 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <scope>provided</scope>
    <version>3.3.2</version>
</dependency>
```

## 配置对@PermitAll接口放行

1. 配置WebSecurityConfigurerAdapter

```java
@AutoConfiguration
@AutoConfigureOrder(-1) // 目的：先于 Spring Security 自动配置，避免一键改包后，org.* 基础包无法生效
@EnableMethodSecurity(securedEnabled = true) // 方法级别安全性控制
public class CustomWebSecurityConfigurerAdapter{

    @Resource
    private ApplicationContext applicationContext;

    /**
     * 在 Spring Security 5.7 之后通过注册SecurityFilterChain的Bean方式替代依赖实现WebSecurityConfigurerAdapter接口的方式
     * @param httpSecurity
     * @return
     * @throws Exception
     */
    @Bean
    protected SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception{
        // 通用配置
        httpSecurity
                // 开启跨域
                .cors(Customizer.withDefaults())
                // CSRF 禁用，因为不使用 Session
                .csrf(AbstractHttpConfigurer::disable);
        // 获得 @PermitAll 带来的 URL 列表，免登录
        Multimap<HttpMethod, String> permitAllUrls = getPermitAllUrlsFromAnnotations();

        httpSecurity
                // ①：全局共享规则
                .authorizeHttpRequests(c ->c
                        // 1.1 静态资源，可匿名访问
                        .requestMatchers(HttpMethod.GET, "/*.html", "/*.html", "/*.css", "/*.js").permitAll()
                        // 1.2 设置 @PermitAll 无需认证
                        .requestMatchers(HttpMethod.GET, permitAllUrls.get(HttpMethod.GET).toArray(new String[0])).permitAll()
                        .requestMatchers(HttpMethod.POST, permitAllUrls.get(HttpMethod.POST).toArray(new String[0])).permitAll()
                        .requestMatchers(HttpMethod.PUT, permitAllUrls.get(HttpMethod.PUT).toArray(new String[0])).permitAll()
                        .requestMatchers(HttpMethod.DELETE, permitAllUrls.get(HttpMethod.DELETE).toArray(new String[0])).permitAll()
                        .requestMatchers(HttpMethod.HEAD, permitAllUrls.get(HttpMethod.HEAD).toArray(new String[0])).permitAll()
                        .requestMatchers(HttpMethod.PATCH, permitAllUrls.get(HttpMethod.PATCH).toArray(new String[0])).permitAll()
                        // 放行 Knife4j 和 Swagger 相关的 URL
                        .requestMatchers(
                                "/swagger-ui/**",
                                "/swagger-resources/**",
                                "/v3/api-docs/**",
                                "/webjars/**",
                                "/doc.html"
                        ).permitAll())
                // ③：兜底规则，必须认证
                .authorizeHttpRequests(c -> c.anyRequest().authenticated());
        return httpSecurity.build();
    }

    /**
     * Multimap:Google 的 Guava 库，提供.类似于 Map，但每个键可以关联多个值
     *
     */
    private Multimap<HttpMethod,String> getPermitAllUrlsFromAnnotations(){
        Multimap<HttpMethod, String> result = HashMultimap.create();
        // 获得接口对应的 HandlerMethod 集合
        RequestMappingHandlerMapping handlerMapping = applicationContext.getBean(RequestMappingHandlerMapping.class);
        // 获取所有的 HandlerMethod 映射关系
        Map<RequestMappingInfo, HandlerMethod> handlerMethods = handlerMapping.getHandlerMethods();
        // 筛选出带有@PermitAll注解的接口
        for (Map.Entry<RequestMappingInfo, HandlerMethod> entry : handlerMethods.entrySet()) {
            RequestMappingInfo requestMappingInfo = entry.getKey();
            HandlerMethod handlerMethod = entry.getValue();
            // 如果该接口没有@PermitAll注解
            if (!handlerMethod.hasMethodAnnotation(PermitAll.class)) continue;

            Set<String> urls = new HashSet<>();
            // 多路径的情况 @PostMapping({"/login","/hello","zef"})
            if (Objects.nonNull(requestMappingInfo.getPatternsCondition()))
                urls.addAll(entry.getKey().getPatternsCondition().getPatterns());

            // 单路径的情况 @PostMapping("/login")
            if (Objects.nonNull(requestMappingInfo.getPathPatternsCondition()))
                urls.addAll(convertList(requestMappingInfo.getPathPatternsCondition().getPatterns(), PathPattern::getPatternString));

            if (urls.isEmpty())continue;

            // 根据请求方法，添加到 result 结果
            requestMappingInfo.getMethodsCondition().getMethods().forEach(requestMethod -> {
                switch (requestMethod) {
                    case GET:
                        result.putAll(HttpMethod.GET, urls);
                        break;
                    case POST:
                        result.putAll(HttpMethod.POST, urls);
                        break;
                    case PUT:
                        result.putAll(HttpMethod.PUT, urls);
                        break;
                    case DELETE:
                        result.putAll(HttpMethod.DELETE, urls);
                        break;
                    case HEAD:
                        result.putAll(HttpMethod.HEAD, urls);
                        break;
                    case PATCH:
                        result.putAll(HttpMethod.PATCH, urls);
                        break;
                }
            });

        }
        return result;
    }
    public static <T, U> List<U> convertList(Collection<T> from, Function<T, U> func) {
        if (CollUtil.isEmpty(from)) {
            return new ArrayList<>();
        }
        return from.stream().map(func).filter(Objects::nonNull).collect(Collectors.toList());
    }
}
```

2. 配置 `spring.factories`


> 具体关于SpringBoot自定义Starter的实现思路

- 创建项目：首先创建一个新 Maven 项目，命名为 `xxx-spring-boot-starter`
- 添加依赖：确保`spring-boot-autoconfigure`依赖已添加，以支持自动配置功能。
- 配置类：编写AutoConfiguration配置类，并使用注解`@Configuration`和`@ConditionlOnMissingBean`来实现有条件加载Bean
- 定义功能：比如要封装一个特定的服务（如短信发送服务），定义接口和实现类，将其逻辑写入`@Service`标注的实现类中
- 自动配置：在`META-INF/spring/org.springframework.boot.autocnfigure.AutoConfiguration.imports`中指定自动配置类的位置。
- 与SpringBoot2.X不同，SpringBoot3.X不再使用sping.factories文件。


3. 完成以上配置 我们可以在Controller类上添加注解将会被Spring Security放行了

```java
@PostMapping("/login")
@PermitAll 
@Operation(summary = "使用账号密码登录")
public CommonResult<AuthLoginRespVO> login(@RequestBody @Valid AuthLoginReqVO reqVO) {
    return success(authService.login(reqVO));
}
```
