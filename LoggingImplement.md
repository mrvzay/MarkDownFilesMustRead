# Implementing Logging in Spring Boot Applications

Spring Boot provides robust logging capabilities out of the box. Here's a comprehensive guide to implementing logging in your application:

## 1. Basic Configuration

### Default Logging (Using application.properties)
```properties
# application.properties
logging.level.root=INFO
logging.level.org.springframework=WARN
logging.level.com.yourpackage=DEBUG

# File logging
logging.file.name=application.log
logging.file.path=/var/logs
logging.file.max-size=10MB
logging.file.max-history=7

# Pattern configuration
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
logging.pattern.file=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
```

## 2. Using SLF4J with Logback (Recommended)

### Add Logback Configuration (logback-spring.xml)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <property name="LOG_PATH" value="${LOG_PATH:-logs}"/>
    <property name="LOG_FILE" value="${LOG_FILE:-application}"/>
    <property name="MAX_FILE_SIZE" value="10MB"/>
    <property name="MAX_HISTORY_DAYS" value="7"/>

    <!-- Console Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %highlight(%-5level) %cyan(%logger{36}) - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- File Appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${LOG_FILE}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${LOG_FILE}-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
            <maxHistory>${MAX_HISTORY_DAYS}</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Loggers -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>

    <!-- Custom package logging -->
    <logger name="com.yourpackage" level="DEBUG"/>
    
    <!-- Special loggers -->
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE"/>
</configuration>
```

## 3. Implementing Logging in Code

### Controller Logging Example
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class UserController {
    
    private static final Logger logger = LoggerFactory.getLogger(UserController.class);
    
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        logger.debug("Request received for user ID: {}", id);
        
        try {
            User user = userService.findById(id);
            logger.info("Successfully retrieved user: {}", user.getId());
            return ResponseEntity.ok(user);
        } catch (UserNotFoundException e) {
            logger.error("User not found with ID: {}", id, e);
            return ResponseEntity.notFound().build();
        }
    }
}
```

### Service Layer Logging
```java
@Service
public class UserService {
    
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);
    
    @Transactional
    public User findById(Long id) {
        logger.debug("Searching for user with ID: {}", id);
        long startTime = System.currentTimeMillis();
        
        User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));
        
        long duration = System.currentTimeMillis() - startTime;
        logger.info("User lookup completed in {} ms", duration);
        
        return user;
    }
}
```

## 4. Advanced Logging Features

### MDC (Mapped Diagnostic Context)
```java
import org.slf4j.MDC;

@RestController
public class OrderController {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderController.class);
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        // Add request ID to MDC
        MDC.put("requestId", UUID.randomUUID().toString());
        
        logger.info("Creating new order for customer: {}", request.getCustomerId());
        
        try {
            Order order = orderService.createOrder(request);
            logger.info("Order created successfully: {}", order.getId());
            return ResponseEntity.ok(order);
        } finally {
            MDC.clear(); // Clean up MDC
        }
    }
}
```

Add to logback.xml to include MDC values:
```xml
<pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] [%X{requestId}] %-5level %logger{36} - %msg%n</pattern>
```

### Conditional Logging
```java
if (logger.isDebugEnabled()) {
    logger.debug("Detailed object state: {}", expensiveToStringOperation());
}
```

## 5. Structured Logging (JSON Format)

### Update logback-spring.xml
```xml
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"application":"${spring.application.name}"}</customFields>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

## 6. External Log Management

### ELK Stack Integration
```properties
# For Logstash integration
logging.logstash.url=logstash:5044
```

### Add Logstash Dependency
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>6.6</version>
</dependency>
```

## 7. Performance Considerations

### Asynchronous Logging
```xml
<configuration>
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>1000</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <includeCallerData>false</includeCallerData>
        <appender-ref ref="FILE"/>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="ASYNC_FILE"/>
    </root>
</configuration>
```

## Best Practices

1. **Use appropriate log levels**:
   - ERROR: System failures
   - WARN: Potentially harmful situations
   - INFO: Important business processes
   - DEBUG: Detailed debug information
   - TRACE: Very detailed tracing

2. **Include context** in log messages (user IDs, request IDs, etc.)

3. **Avoid sensitive data** in logs (passwords, PII)

4. **Use parameterized messages** instead of string concatenation:
   ```java
   // Good
   logger.debug("User {} logged in", userId);
   
   // Bad
   logger.debug("User " + userId + " logged in");
   ```

5. **Regularly review log levels** - increase severity for production

6. **Implement log rotation** to prevent disk space issues

This comprehensive logging implementation will give you visibility into your Spring Boot application's behavior while maintaining good performance and manageability.


------------------------------


Implementing a **logging mechanism** in a Spring Boot application is essential for **debugging**, **monitoring**, and **auditing**. Spring Boot comes with **built-in support for logging** using **SLF4J** with **Logback** (default), but you can customize it easily.

---

## ✅ 1. **Default Logging in Spring Boot**

Spring Boot includes:

* **SLF4J** API
* **Logback** as the default logging implementation

### Usage:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@RestController
public class UserController {

    private static final Logger logger = LoggerFactory.getLogger(UserController.class);

    @GetMapping("/hello")
    public String hello() {
        logger.info("INFO: /hello endpoint called");
        logger.debug("DEBUG: Detailed info for debugging");
        return "Hello World";
    }
}
```

---

## ✅ 2. **Set Log Levels in `application.properties` or `application.yml`**

### `application.properties`

```properties
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.file.name=logs/app.log
```

### `application.yml`

```yaml
logging:
  level:
    root: INFO
    com.example: DEBUG
  file:
    name: logs/app.log
```

---

## ✅ 3. **Logging Levels (in increasing detail)**

| Level | Purpose                                  |
| ----- | ---------------------------------------- |
| ERROR | Critical issues causing failure          |
| WARN  | Warnings about potential problems        |
| INFO  | General app flow information             |
| DEBUG | Detailed debugging info (dev mode)       |
| TRACE | Most detailed logs (method-level traces) |

---

## ✅ 4. **Logback Configuration (Advanced Customization)**

Create a `logback-spring.xml` in `src/main/resources`:

```xml
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>logs/custom.log</file>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

> ✅ Use `logback-spring.xml` instead of `logback.xml` to enable Spring profile support.

---

## ✅ 5. **Logging Request & Response (Optional with Filter/Aspect)**

### Using Filter:

```java
@Component
public class LoggingFilter implements Filter {

    private static final Logger logger = LoggerFactory.getLogger(LoggingFilter.class);

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest req = (HttpServletRequest) request;
        logger.info("Incoming Request: {} {}", req.getMethod(), req.getRequestURI());
        chain.doFilter(request, response);
    }
}
```

---

## ✅ 6. **Aspect-Oriented Logging (AOP) for Service/Repository Methods**

```java
@Aspect
@Component
public class LoggingAspect {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        logger.info("Calling method: {}", joinPoint.getSignature().getName());
    }
}
```

---

## ✅ 7. **External Logging Tools Integration**

You can also integrate Spring Boot with:

* **ELK Stack** (Elasticsearch + Logstash + Kibana)
* **Graylog**, **Splunk**, or **Datadog**
* **Fluentd**, **Grafana Loki**

---

## ✅ Summary: Steps to Implement Logging

| Step                          | Description                 |
| ----------------------------- | --------------------------- |
| Use SLF4J Logger              | `LoggerFactory.getLogger()` |
| Set levels in application.yml | `logging.level.*`           |
| Customize output              | `logback-spring.xml`        |
| Log requests/responses        | Use `Filter` or `@Aspect`   |
| External tools integration    | ELK, Datadog, Grafana, etc. |

---

Would you like a full working example showing:

* Request logging with filters?
* Method logging with AOP?
* Logback rolling file config?

Let me know your preferred direction!
