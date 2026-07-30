# 官方版本与事实来源

> 教材正文以初学者教学为主，本页集中保留版本事实，避免每题被链接打断。

## Java / OpenJDK

- JDK 25 项目页：<https://openjdk.org/projects/jdk/25/>
- JEP 444 Virtual Threads：<https://openjdk.org/jeps/444>
- JEP 491 Synchronize Virtual Threads without Pinning：<https://openjdk.org/jeps/491>
- JEP 374 Deprecate and Disable Biased Locking：<https://openjdk.org/jeps/374>
- Java 25 GC Tuning Guide：<https://docs.oracle.com/en/java/javase/25/gctuning/>
- Java 25 可用收集器：<https://docs.oracle.com/en/java/javase/25/gctuning/available-collectors.html>
- Java 25 ZGC：<https://docs.oracle.com/en/java/javase/25/gctuning/z-garbage-collector.html>

## Spring Framework / Spring Boot

- Spring Framework Dependency Injection：
  <https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html>
- Spring Boot System Requirements：
  <https://docs.spring.io/spring-boot/system-requirements.html>
- Spring Boot Auto-configuration：
  <https://docs.spring.io/spring-boot/reference/using/auto-configuration.html>
- Creating Your Own Auto-configuration：
  <https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html>

## 本教材据此修正的旧说法

1. JDK 25 已于 2025-09-16 GA，并被多数厂商作为 LTS。
2. 虚拟线程在 Java 21 正式交付，不是实验功能。
3. JDK 24 的 JEP 491 让虚拟线程在 `synchronized` 中阻塞时不再出现绝大多数旧式 pinning。
4. 偏向锁从 JDK 15 起默认禁用，不能作为 Java 21/25 的固定锁升级第一阶段。
5. 多数现代 HotSpot 环境默认使用 G1；ZGC 以极低暂停为目标但有吞吐和资源取舍。
6. Spring Boot 4.1 要求 Java 17+，兼容到 Java 26。
7. 现代 Spring Boot 自动配置候选使用
   `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。
8. 构造器循环依赖不可解析；setter 循环即使框架有机制，也不应作为推荐架构。
