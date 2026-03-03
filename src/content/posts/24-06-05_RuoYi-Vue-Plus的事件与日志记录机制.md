---
title: RuoYi-Vue-Plus 的事件与日志记录机制详解
published: 2024-06-05
description: RuoYi-Vue-Plus 的事件与日志记录机制详解
tags: [java, spring, ruoyi]
category: 后端
---

## 一、事件机制简介

事件机制是 Spring 框架中的一种设计模式，它允许应用程序在运行时动态地发布和订阅事件。这种机制的核心思想是将事件的发布者（Publisher）和事件的订阅者（Subscriber）解耦，使得两者之间不需要直接的依赖关系。在 RuoYi-Vue-Plus 中，事件机制被用于记录登录信息、操作日志等场景。

### 1. 事件对象

事件对象是事件机制中的核心，它封装了事件的所有相关信息。`LogininforEvent` 是用于记录登录信息的事件对象,代码示例：

```java
@Data
public class LogininforEvent {
    private String username;  // 用户名
    private String status;   // 登录状态（success/fail/kickout 等）
    private String message;  // 登录消息（如错误信息）
    private HttpServletRequest request;  // 当前请求对象
    private Object[] args;
}
```

### 2. 事件发布

在业务代码中，当需要记录登录信息时，会创建一个 `LogininforEvent` 对象，并通过 Spring 的事件发布机制将其发布出去。以下是事件发布的代码示例：

```java
private void recordLogininfor(String username, String status, String message) {
    LogininforEvent event = new LogininforEvent();
    event.setUsername(username);
    event.setStatus(status);
    event.setMessage(message);
    event.setRequest(ServletUtils.getRequest());  // 获取当前请求对象
    SpringUtils.context().publishEvent(event);    // 发布事件
}
```

这段代码中，`SpringUtils.context().publishEvent(event)` 是将事件发布到 Spring 的事件广播器中的关键语句。一旦事件被发布，Spring 会自动通知所有订阅了该事件的监听器。

### 3. 事件监听

事件监听器是事件机制中的订阅者，它负责处理事件。在 RuoYi-Vue-Plus 中，`UserActionListener` 是默认监听器，它监听 `LogininforEvent` 并进行相应的处理。代码示例：
```java
@Component
public class UserActionListener {
    @EventListener
    @Async("asyncExecutor")  // 指定异步线程池
    public void handleLogininfor(LogininforEvent event) {
        // 组装 SysLogininfor 实体
        SysLogininfor log = new SysLogininfor();
        log.setUsername(event.getUsername());
        log.setStatus(event.getStatus());
        log.setMessage(event.getMessage());
        log.setIp(ServletUtils.getClientIP(event.getRequest()));  // 获取客户端 IP
        log.setLoginLocation(AddressUtils.getRealAddressByIP(log.getIp()));  // 获取登录地点
        log.setBrowser(ServletUtils.getBrowser(event.getRequest()));  // 获取浏览器信息
        log.setOs(ServletUtils.getOs(event.getRequest()));  // 获取操作系统信息
        log.setCreateTime(new Date());  // 设置创建时间

        // 调用服务层将日志信息保存到数据库
        logininforService.insertLogininfor(log);

        // 刷新 Redis 中的在线用户信息
        onlineUserService.refresh(event.getUsername());
    }
}
```

在这个监听器中：
- `@EventListener` 注解表示该方法会监听 `LogininforEvent` 事件。
- `@Async("asyncExecutor")` 注解则表示该方法会在一个异步线程中执行，避免阻塞主线程。
- `logininforService.insertLogininfor(log)` 是将登录信息持久化到数据库中的操作。
- `onlineUserService.refresh(event.getUsername())` 是刷新 Redis 中的在线用户信息。

### 4. 异步线程池配置

为了支持异步处理，RuoYi-Vue-Plus 配置了一个专门的线程池。线程池的配置代码：

```java
@Configuration
public class ThreadPoolConfig {
    @Bean("asyncExecutor")
    public ThreadPoolTaskExecutor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);  // 核心线程数
        executor.setMaxPoolSize(32);  // 最大线程数
        executor.setQueueCapacity(200);  // 队列容量
        executor.setThreadNamePrefix("log-async-");  // 线程名称前缀
        executor.initialize();
        return executor;
    }
}
```

通过配置线程池，可以合理分配资源，避免线程过多导致的资源竞争问题。

## 二、日志记录的优势

相比传统的直接使用 `log.info` 的日志记录方式，RuoYi-Vue-Plus 中基于事件机制的日志记录具有以下优势：

### 1. 解耦与职责分离

在传统的日志记录方式中，业务代码与日志记录紧密耦合。如果需要修改日志格式或存储方式，必须修改业务代码。而在事件机制中，业务代码只负责发布事件，具体的日志记录逻辑由监听器实现。这种方式使得业务代码与日志记录逻辑完全解耦，符合单一职责原则，降低了代码的复杂度，提高了可维护性。

### 2. 异步处理与性能优化

传统的日志记录通常是同步操作，会阻塞当前线程，尤其是在日志写入数据库或远程存储时，可能会影响系统的性能。而在事件机制中，可以通过 `@Async` 注解将日志记录操作放入异步线程池中执行，避免阻塞主线程，从而提升系统的响应速度和性能。

### 3. 动态扩展与灵活性

在传统的日志记录方式中，如果需要增加新的日志处理逻辑，必须修改业务代码。而在事件机制中，只需新增一个监听器即可实现新的日志处理逻辑，无需修改业务代码。这种方式使得日志记录逻辑具有极高的灵活性和可扩展性，可以轻松地适应不同的业务需求。

## 三、实际应用场景

### 1. 登录日志记录

在用户登录时，系统会调用 `recordLogininfor` 方法发布一个 `LogininforEvent` 事件。`UserActionListener` 监听到该事件后，会将登录信息持久化到数据库中。这种方式不仅可以记录用户的登录状态和时间，还可以通过扩展监听器将登录信息发送到消息队列、Elasticsearch 等其他存储系统中，方便后续的分析和监控。

### 2. 操作日志记录

除了登录日志，RuoYi-Vue-Plus 还支持操作日志的记录。当用户执行某些操作（如添加、删除、修改数据）时，系统会发布一个 `OperLogEvent` 事件。监听器会将操作日志记录到数据库中，便于后续的审计和问题排查。

以下是 `OperLogEvent` 的代码示例：

```java
@Data
public class OperLogEvent {
    private String operName;  // 操作人
    private String operUrl;  // 操作 URL
    private String operIp;   // 操作 IP
    private String operMethod;  // 操作方法
    private String operParam;   // 操作参数
    private String status;   // 操作状态
    private String message;  // 操作消息
    private Long operTime;   // 操作时间

}
```

以下是操作日志的事件发布代码：

```java
private void recordOperLog(String operName, String operUrl, String operIp, String operMethod, String operParam, String status, String message) {
    OperLogEvent event = new OperLogEvent();
    event.setOperName(operName);
    event.setOperUrl(operUrl);
    event.setOperIp(operIp);
    event.setOperMethod(operMethod);
    event.setOperParam(operParam);
    event.setStatus(status);
    event.setMessage(message);
    event.setOperTime(System.currentTimeMillis());
    SpringUtils.context().publishEvent(event);
}
```

以下是操作日志的事件监听器代码：

```java
@Component
public class OperLogListener {
    @EventListener
    @Async("asyncExecutor")
    public void handleOperLog(OperLogEvent event) {
        SysOperLog log = new SysOperLog();
        log.setOperName(event.getOperName());
        log.setOperUrl(event.getOperUrl());
        log.setOperIp(event.getOperIp());
        log.setOperMethod(event.getOperMethod());
        log.setOperParam(event.getOperParam());
        log.setStatus(event.getStatus());
        log.setMessage(event.getMessage());
        log.setOperTime(new Date(event.getOperTime()));
        operLogService.insertOperLog(log);
    }
}
```


## 四、动态扩展与灵活性

在事件机制中，只需新增一个监听器即可实现新的日志处理逻辑，无需修改业务代码。例如，如果需要将日志发送到 Elasticsearch，可以新增一个监听器：

```java
@Component
public class EsLogininforListener {
    @EventListener
    @Async("asyncExecutor")
    public void pushToEs(LogininforEvent event) {
        // 构造 Elasticsearch 日志对象
        EsLogininfor esLog = new EsLogininfor();
        esLog.setUsername(event.getUsername());
        esLog.setStatus(event.getStatus());
        esLog.setMessage(event.getMessage());
        esLog.setIp(ServletUtils.getClientIP(event.getRequest()));
        esLog.setLoginLocation(AddressUtils.getRealAddressByIP(esLog.getIp()));
        esLog.setBrowser(ServletUtils.getBrowser(event.getRequest()));
        esLog.setOs(ServletUtils.getOs(event.getRequest()));
        esLog.setCreateTime(new Date());
        // 发送到 Elasticsearch
        esLogService.save(esLog);
    }
}
```
