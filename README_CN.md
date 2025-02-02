# Spring REST 异常处理实现指南

## 目录
1. [快速开始](#1-快速开始)
2. [完整功能实现](#2-完整功能实现)
3. [高级特性](#3-高级特性)
4. [最佳实践](#4-最佳实践)
5. [历史分析与设计思路](#5-历史分析与设计思路)
6. [常见问题](#6-常见问题)

## 1. 快速开始

### 1.1 环境要求
- Spring Boot 2.7.0 或更高版本（推荐 3.0+ 使用 ProblemDetail）
- Java 17 或更高版本（使用 Spring Boot 3.0+）
- Maven 或 Gradle

### 1.2 基础依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 1.3 最简实现
1. 添加全局异常处理器：
```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    @ExceptionHandler(Exception.class)
    ProblemDetail handleAllExceptions(Exception ex) {
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            ex.getMessage()
        );
    }
}
```

2. 启用 Problem Details 支持：
```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

## 2. 完整功能实现

### 2.1 错误码定义
```java
public interface ErrorCodes {
    // 通用错误 (1000-1999)
    int UNKNOWN_ERROR = 1000;
    int VALIDATION_ERROR = 1001;
    int ACCESS_DENIED = 1002;
    
    // 业务错误 (2000-2999)
    int USER_NOT_FOUND = 2000;
    int ORDER_NOT_FOUND = 2001;
}
```

### 2.2 配置文件
```yaml
spring:
  mvc:
    problemdetails:
      enabled: true  # 启用 RFC 7807 Problem Details 支持
  jackson:
    default-property-inclusion: non_null  # 序列化时忽略 null 值
    serialization:
      indent-output: true  # 美化输出的 JSON（开发环境推荐）
      write-dates-as-timestamps: false  # 使用 ISO-8601 格式序列化日期

api:
  error:
    more-info-url-template: "https://api.example.com/docs/errors/%s"
    include-stacktrace: never  # 可选值：ALWAYS, NEVER, ON_PARAM
    include-developer-message: false
```

### 2.3 配置属性类
```java
@Configuration
@ConfigurationProperties(prefix = "api.error")
@Getter
@Setter
public class ErrorProperties {
    private String moreInfoUrlTemplate = "https://api.example.com/errors/%s";
    private StackTraceMode includeStacktrace = StackTraceMode.NEVER;
    private boolean includeDeveloperMessage = false;
    
    public enum StackTraceMode {
        ALWAYS, NEVER, ON_PARAM
    }
}
```

### 2.4 错误响应构建器
```java
@Component
@RequiredArgsConstructor
public class ProblemDetailBuilder {
    private final ErrorProperties errorProperties;
    
    public ProblemDetail build(
            HttpStatus status, 
            int errorCode,
            String message,
            Map<String, Object> properties) {
            
        ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(
            status,
            message
        );
        
        // 设置基本属性
        problemDetail.setType(URI.create(
            String.format(errorProperties.getMoreInfoUrlTemplate(), errorCode)
        ));
        problemDetail.setProperty("code", errorCode);
        problemDetail.setProperty("timestamp", LocalDateTime.now());
        
        // 根据配置添加开发者信息
        if (errorProperties.isIncludeDeveloperMessage()) {
            problemDetail.setProperty("developerMessage", message);
        }
        
        // 添加自定义属性
        if (properties != null) {
            properties.forEach(problemDetail::setProperty);
        }
        
        return problemDetail;
    }
}
```

### 2.5 全局异常处理器
```java
@RestControllerAdvice
@RequiredArgsConstructor
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    private final ProblemDetailBuilder problemDetailBuilder;
    
    // 处理业务异常
    @ExceptionHandler(BusinessException.class)
    ProblemDetail handleBusinessException(BusinessException ex) {
        return problemDetailBuilder.build(
            HttpStatus.BAD_REQUEST,
            ErrorCodes.BUSINESS_ERROR,
            ex.getMessage(),
            Map.of("businessCode", ex.getBusinessCode())
        );
    }
    
    // 处理资源未找到
    @ExceptionHandler(ResourceNotFoundException.class)
    ProblemDetail handleResourceNotFound(ResourceNotFoundException ex) {
        return problemDetailBuilder.build(
            HttpStatus.NOT_FOUND,
            ErrorCodes.USER_NOT_FOUND,
            ex.getMessage(),
            Map.of("resourceId", ex.getResourceId())
        );
    }
    
    // 处理其他异常
    @ExceptionHandler(Exception.class)
    ProblemDetail handleOthers(Exception ex) {
        return problemDetailBuilder.build(
            HttpStatus.INTERNAL_SERVER_ERROR,
            ErrorCodes.UNKNOWN_ERROR,
            "An unexpected error occurred",
            null
        );
    }
}
```

### 2.6 使用示例
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable String id) {
        User user = userService.findById(id);
        if (user == null) {
            throw new ResourceNotFoundException("User not found", id);
        }
        return user;
    }
}

// 自定义异常
@Getter
public class ResourceNotFoundException extends RuntimeException {
    private final String resourceId;
    
    public ResourceNotFoundException(String message, String resourceId) {
        super(message);
        this.resourceId = resourceId;
    }
}
```

## 3. 高级特性

### 3.1 异常映射定义
对于复杂项目，可以使用更灵活的异常映射机制：

```java
@Configuration
public class ErrorHandlingConfig {
    
    /**
     * 异常映射定义，支持通配符匹配和继承关系
     */
    @Bean
    public ExceptionMappingDefinitions exceptionMappingDefinitions() {
        Map<String, ErrorDefinition> definitions = new HashMap<>();
        
        // 基础错误定义
        definitions.put("java.lang.Throwable",
            new ErrorDefinition(HttpStatus.INTERNAL_SERVER_ERROR, ErrorCodes.UNKNOWN_ERROR));
            
        // 通用业务异常
        definitions.put("com.example.common.BusinessException",
            new ErrorDefinition(HttpStatus.BAD_REQUEST, ErrorCodes.BUSINESS_ERROR));
            
        // 访问控制
        definitions.put("org.springframework.security.access.AccessDeniedException",
            new ErrorDefinition(HttpStatus.FORBIDDEN, ErrorCodes.ACCESS_DENIED));
            
        // 验证错误
        definitions.put("javax.validation.ValidationException",
            new ErrorDefinition(HttpStatus.BAD_REQUEST, ErrorCodes.VALIDATION_ERROR));
            
        // 支持模块化的异常匹配
        definitions.put("com.example.user.*Exception",
            new ErrorDefinition(HttpStatus.BAD_REQUEST, ErrorCodes.USER_ERROR));
            
        return new ExceptionMappingDefinitions(definitions);
    }
}

/**
 * 错误定义，包含 HTTP 状态码和业务错误码
 */
@Value
public class ErrorDefinition {
    HttpStatus status;
    int errorCode;
    String messageTemplate;
    String moreInfoUrlTemplate;
}

/**
 * 异常映射定义管理器
 */
@RequiredArgsConstructor
public class ExceptionMappingDefinitions {
    private final Map<String, ErrorDefinition> definitions;
    
    public ErrorDefinition resolve(Throwable ex) {
        String exClassName = ex.getClass().getName();
        
        // 1. 精确匹配
        ErrorDefinition definition = definitions.get(exClassName);
        if (definition != null) {
            return definition;
        }
        
        // 2. 通配符匹配
        for (Map.Entry<String, ErrorDefinition> entry : definitions.entrySet()) {
            String pattern = entry.getKey();
            if (pattern.endsWith("*") && exClassName.startsWith(pattern.substring(0, pattern.length() - 1))) {
                return entry.getValue();
            }
        }
        
        // 3. 继承关系匹配
        Class<?> exClass = ex.getClass();
        while (exClass != Object.class) {
            definition = definitions.get(exClass.getName());
            if (definition != null) {
                return definition;
            }
            exClass = exClass.getSuperclass();
        }
        
        // 4. 默认为 Throwable 的定义
        return definitions.get("java.lang.Throwable");
    }
}
```

### 3.2 国际化支持
完整的国际化支持实现：

```java
@Configuration
@ConditionalOnProperty(prefix = "api.error", name = "messages.basename")
public class MessageSourceConfig {
    
    @Bean
    public MessageSource errorMessageSource(ErrorProperties properties) {
        ReloadableResourceBundleMessageSource messageSource = 
            new ReloadableResourceBundleMessageSource();
        messageSource.setBasename(properties.getMessagesBasename());
        messageSource.setDefaultLocale(properties.getDefaultLocale());
        messageSource.setDefaultEncoding("UTF-8");
        messageSource.setCacheSeconds(properties.getMessageCacheSeconds());
        return messageSource;
    }
}
```

```properties
# 消息资源文件
# messages/errors_zh_CN.properties
error.2000=用户未找到：{0}
error.2001=用户已存在：{0}
error.3001=库存不足，当前库存：{0}，需求数量：{1}

# messages/errors_en.properties
error.2000=User not found: {0}
error.2001=User already exists: {0}
error.3001=Insufficient stock, current: {0}, required: {1}
```

### 3.3 方案对比分析

1. **直接处理方案**
   优点：
   - 简单直观，代码量少
   - 每个异常处理方法都很清晰
   - 编译时类型安全
   
   缺点：
   - 异常处理逻辑分散
   - 需要为每种异常写专门的处理方法
   - 配置不够灵活

2. **映射定义方案**
   优点：
   - 支持灵活的异常匹配
   - 配置集中且易于修改
   - 运行时动态性强
   
   缺点：
   - 代码相对复杂
   - 基于字符串的配置
   - 可能有性能开销

3. **混合策略（推荐）**
   优点：
   - 关键异常使用直接处理
   - 通用异常使用映射配置
   - 平衡了灵活性和简洁性
   
   实现示例见 5.2.5 节的 `GlobalExceptionHandler`

## 4. 最佳实践

### 4.1 方案选择
1. **项目规模与方案对应**
   - 小型项目：使用基础实现（5.1节）
   - 中型项目：使用完整功能实现（5.2节）
   - 大型项目：考虑高级特性（5.3节）

2. **异常处理策略**
   - 业务异常：使用专门的异常类和处理方法
   - 框架异常：使用异常映射配置
   - 未知异常：提供友好的默认处理

### 4.2 设计规范
1. **错误码设计**
   - 1xxx：系统级错误（如：1001-参数验证失败）
   - 2xxx：用户模块错误（如：2001-用户未找到）
   - 3xxx：订单模块错误（如：3001-库存不足）
   - 9xxx：其他未分类错误

2. **异常类设计**
   ```java
   @Getter
   public abstract class BaseException extends RuntimeException {
       private final int errorCode;
       private final Map<String, Object> properties;
       
       protected BaseException(int errorCode, String message) {
           super(message);
           this.errorCode = errorCode;
           this.properties = new HashMap<>();
       }
   }
   
   public class UserNotFoundException extends BaseException {
       public UserNotFoundException(String userId) {
           super(ErrorCodes.USER_NOT_FOUND, "User not found: " + userId);
           getProperties().put("userId", userId);
       }
   }
   ```

### 4.3 环境配置
1. **开发环境**
   ```yaml
   api:
     error:
       include-stacktrace: ALWAYS
       include-developer-message: true
       indent-output: true
   ```

2. **生产环境**
   ```yaml
   api:
     error:
       include-stacktrace: NEVER
       include-developer-message: false
       indent-output: false
   ```

### 4.4 安全考虑
1. **信息隐藏**
   - 生产环境中隐藏技术细节
   - 避免在错误消息中包含敏感信息
   - 使用适当的抽象级别描述错误

2. **访问控制**
   - 错误详情页面需要权限控制
   - 开发者信息只对内部系统可见
   - 堆栈跟踪信息按需显示

### 4.5 性能优化
1. **消息源配置**
   ```java
   @Bean
   public MessageSource messageSource() {
       ReloadableResourceBundleMessageSource messageSource = 
           new ReloadableResourceBundleMessageSource();
       messageSource.setCacheSeconds(3600); // 生产环境缓存1小时
       messageSource.setUseCodeAsDefaultMessage(true); // 避免消息未找到异常
       return messageSource;
   }
   ```

2. **异步日志**
   ```java
   @Async
   public void logError(Exception ex, ProblemDetail problemDetail) {
       // 异步记录错误日志
   }
   ```

### 4.6 测试策略
1. **单元测试**
   ```java
   @Test
   void whenUserNotFound_thenReturn404() {
       Exception ex = new UserNotFoundException("test-id");
       ProblemDetail problem = handler.handleUserNotFoundException(ex);
       
       assertThat(problem.getStatus()).isEqualTo(404);
       assertThat(problem.getProperty("code")).isEqualTo(2001);
   }
   ```

2. **集成测试**
   ```java
   @SpringBootTest
   @AutoConfigureMockMvc
   class ErrorHandlingIntegrationTest {
       @Test
       void whenInvalidRequest_thenReturn400() {
           mockMvc.perform(get("/api/users/invalid-id"))
               .andExpect(status().isNotFound())
               .andExpect(jsonPath("$.code").value(2001));
       }
   }
   ```

### 4.7 维护建议
1. **文档维护**
   - 及时更新错误码文档
   - 记录异常处理的变更历史
   - 提供错误码查询接口

2. **代码审查**
   - 确保异常处理的一致性
   - 检查错误消息的准确性
   - 验证安全性要求的遵守

3. **监控告警**
   - 记录异常发生频率
   - 设置关键错误的告警阈值
   - 定期分析错误趋势

## 5. 历史分析与设计思路

### 5.1 错误响应的标准化
`RestError` 的设计理念值得借鉴：
- 统一的错误响应格式
- 包含 status、code、message、developerMessage 等字段
- 支持更多错误详情的扩展

### 5.2 错误解析的分层设计
接口设计思路仍然有参考价值：
```java
public interface RestErrorResolver
public interface RestErrorConverter<T>
```

## 6. 常见问题

### 6.1 先决条件说明
- Spring Boot 2.7.0 或更高版本（推荐 3.0+ 使用 ProblemDetail）
- Java 17 或更高版本（使用 Spring Boot 3.0+）
- Maven 或 Gradle

### 6.2 配置项说明
- `spring.mvc.problemdetails.enabled`: 启用 RFC 7807 Problem Details 支持
- `spring.jackson.default-property-inclusion`: 序列化时忽略 null 值
- `spring.jackson.serialization.indent-output`: 美化输出的 JSON（开发环境推荐）
- `spring.jackson.serialization.write-dates-as-timestamps`: 使用 ISO-8601 格式序列化日期
- `spring.jackson.serialization.fail-on-empty-beans`: 序列化时忽略空对象
- `spring.jackson.deserialization.fail-on-unknown-properties`: 反序列化时忽略未知属性
- `api.error.more-info-url-template`: 错误信息URL模板
- `api.error.include-stacktrace`: 堆栈跟踪显示模式
- `api.error.include-developer-message`: 是否包含开发者信息
- `api.error.messages.basename`: 消息资源文件基础名
- `api.error.messages.default-locale`: 默认语言

### 6.3 常见问题解答
1. **如何选择合适的异常处理方案？**
   - 对于小型项目，直接处理方案可能更简单。
   - 对于中型项目，完整功能实现可以提供更灵活的异常处理。
   - 对于大型项目，高级特性可以提供更复杂的异常处理。

2. **如何设计错误码？**
   - 1xxx：系统级错误（如：1001-参数验证失败）
   - 2xxx：用户模块错误（如：2001-用户未找到）
   - 3xxx：订单模块错误（如：3001-库存不足）
   - 9xxx：其他未分类错误

3. **如何设计异常类？**
   - 使用 `@Getter` 注解创建一个抽象的 `BaseException` 类
   - 为每个具体的异常创建一个子类，并调用父类的构造函数

4. **如何配置环境？**
   - 开发环境：`api.error.include-stacktrace: ALWAYS` 和 `api.error.include-developer-message: true`
   - 生产环境：`api.error.include-stacktrace: NEVER` 和 `api.error.include-developer-message: false`

5. **如何处理性能问题？**
   - 配置消息源缓存时间
   - 使用异步日志记录错误

6. **如何进行测试？**
   - 编写单元测试
   - 编写集成测试

7. **如何维护代码？**
   - 及时更新错误码文档
   - 记录异常处理的变更历史
   - 提供错误码查询接口
