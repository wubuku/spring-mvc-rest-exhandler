# Spring REST 异常处理实现指南

## 目录
- [Spring REST 异常处理实现指南](#spring-rest-异常处理实现指南)
  - [目录](#目录)
  - [1. 快速开始](#1-快速开始)
    - [1.1 环境要求](#11-环境要求)
    - [1.2 基础依赖](#12-基础依赖)
    - [1.3 最简实现](#13-最简实现)
    - [1.4 Spring Boot 默认异常处理](#14-spring-boot-默认异常处理)
    - [1.5 下一步](#15-下一步)
  - [2. 完整功能实现](#2-完整功能实现)
    - [2.1 错误码定义](#21-错误码定义)
    - [2.2 配置文件](#22-配置文件)
    - [2.3 配置属性类](#23-配置属性类)
    - [2.4 错误响应构建器](#24-错误响应构建器)
    - [2.5 错误响应示例](#25-错误响应示例)
    - [2.6 全局异常处理器](#26-全局异常处理器)
    - [2.7 使用示例](#27-使用示例)
  - [3. 高级特性](#3-高级特性)
    - [3.1 异常映射定义](#31-异常映射定义)
    - [3.2 国际化支持](#32-国际化支持)
    - [3.3 方案对比分析](#33-方案对比分析)
  - [4. 最佳实践](#4-最佳实践)
    - [4.1 方案选择](#41-方案选择)
    - [4.2 设计规范](#42-设计规范)
    - [4.3 环境配置](#43-环境配置)
    - [4.4 安全考虑](#44-安全考虑)
    - [4.5 性能优化](#45-性能优化)
    - [4.6 测试策略](#46-测试策略)
    - [4.7 维护建议](#47-维护建议)
    - [4.8 设计决策说明](#48-设计决策说明)

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
    <version>3.2.0</version>
</dependency>
```

### 1.3 最简实现
1. 添加全局异常处理器：
```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    // 处理未捕获的异常
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

### 1.4 Spring Boot 默认异常处理
通过继承 `ResponseEntityExceptionHandler`，以下异常会自动得到处理：

1. **HTTP 请求相关**
   - `HttpRequestMethodNotSupportedException` (405 Method Not Allowed)
   - `HttpMediaTypeNotSupportedException` (415 Unsupported Media Type)
   - `HttpMediaTypeNotAcceptableException` (406 Not Acceptable)
   - `MissingPathVariableException` (500 Internal Server Error)
   - `MissingServletRequestParameterException` (400 Bad Request)
   - `ServletRequestBindingException` (400 Bad Request)
   - `MethodArgumentNotValidException` (400 Bad Request)

2. **Spring Security 相关**（需添加 spring-boot-starter-security）
   - `AccessDeniedException` (403 Forbidden)
   - `AuthenticationException` (401 Unauthorized)

3. **数据绑定相关**
   - `BindException` (400 Bad Request)
   - `TypeMismatchException` (400 Bad Request)

4. **其他常见异常**
   - `NoHandlerFoundException` (404 Not Found)
   - `AsyncRequestTimeoutException` (503 Service Unavailable)

示例响应：
```json
{
    "type": "about:blank",
    "title": "Not Found",
    "status": 404,
    "detail": "No handler found for GET /unknown-path",
    "instance": "/unknown-path"
}
```

如果需要自定义这些异常的处理，可以重写相应的方法：
```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    @Override
    protected ResponseEntity<Object> handleHttpRequestMethodNotSupported(
            HttpRequestMethodNotSupportedException ex, 
            HttpHeaders headers, 
            HttpStatusCode status, 
            WebRequest request) {
        
        ProblemDetail body = ProblemDetail.forStatusAndDetail(
            HttpStatus.METHOD_NOT_ALLOWED,
            "Method " + ex.getMethod() + " is not supported"
        );
        body.setProperty("supportedMethods", ex.getSupportedHttpMethods());
        
        return new ResponseEntity<>(body, headers, status);
    }
}
```

### 1.5 下一步
上面的最简实现已经能处理基本的异常情况。如果需要：
- 自定义错误码
- 统一的错误响应格式
- 更灵活的异常处理

请参考第2章的完整功能实现。

## 2. 完整功能实现

### 2.1 错误码定义
```java
public interface ErrorCodes {
    // 通用错误 (1000-1999)
    int UNKNOWN_ERROR = 1000;
    int VALIDATION_ERROR = 1001;
    int ACCESS_DENIED = 1002;
    int AUTHENTICATION_FAILED = 1003;
    
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
      enabled: true  # 启用 RFC 7807 规范的错误响应格式
  jackson:
    default-property-inclusion: non_null  # 响应中不包含 null 值的字段
    serialization:
      indent-output: true  # 开发环境下格式化 JSON 输出
      write-dates-as-timestamps: false  # 日期格式使用 ISO-8601

api:
  error:
    # 错误类型前缀，用于标识错误类型
    type-prefix: "https://api.example.com/problems"
    # 错误文档URL，可选配置
    docs-url-template: "https://api.example.com/docs/errors/%s"
    include-stacktrace: never
```

### 2.3 配置属性类
```java
@Configuration
@ConfigurationProperties(prefix = "api.error")
@Getter
@Setter
public class ErrorProperties {
    private String typePrefix;
    private String docsUrlTemplate;
    private StackTraceMode includeStacktrace = StackTraceMode.NEVER;
    
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
        
        // 设置错误类型
        String typePrefix = errorProperties.getTypePrefix();
        if (StringUtils.hasText(typePrefix)) {
            problemDetail.setType(URI.create(
                String.format("%s/%s", typePrefix, errorCode)
            ));
        }
        
        // 设置文档链接（如果配置了）
        String docsUrl = errorProperties.getDocsUrlTemplate();
        if (StringUtils.hasText(docsUrl)) {
            problemDetail.setProperty("moreInfo", 
                String.format(docsUrl, errorCode));
        }
        
        // 设置其他属性
        problemDetail.setProperty("code", errorCode);
        problemDetail.setProperty("timestamp", LocalDateTime.now());
        
        // 添加自定义属性
        if (properties != null) {
            properties.forEach(problemDetail::setProperty);
        }
        
        return problemDetail;
    }
}
```

### 2.5 错误响应示例
1. 完整配置的响应（开发环境）：
```json
{
    "type": "https://api.example.com/problems/2000",
    "title": "Not Found",
    "status": 404,
    "detail": "User with ID '123' not found",
    "code": 2000,
    "moreInfo": "https://api.example.com/docs/errors/2000",
    "timestamp": "2024-01-20T10:15:30.123Z",
    "stackTrace": "com.example.ResourceNotFoundException: User not found\n\tat com.example.UserController.getUser(UserController.java:25)\n..."
}
```

2. 最小配置的响应（生产环境）：
```json
{
    "title": "Not Found",
    "status": 404,
    "detail": "User with ID '123' not found",
    "code": 2000,
    "timestamp": "2024-01-20T10:15:30.123Z"
}
```

### 2.6 全局异常处理器
```java
@RestControllerAdvice
@RequiredArgsConstructor
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    private final ProblemDetailBuilder problemDetailBuilder;
    private final ErrorProperties errorProperties;
    
    private boolean shouldIncludeStackTrace(WebRequest request) {
        StackTraceMode mode = errorProperties.getIncludeStacktrace();
        if (mode == StackTraceMode.ALWAYS) {
            return true;
        }
        if (mode == StackTraceMode.ON_PARAM && request.getParameter("trace") != null) {
            return true;
        }
        return false;
    }
    
    // 处理业务异常
    @ExceptionHandler(BusinessException.class)
    ProblemDetail handleBusinessException(BusinessException ex, WebRequest request) {
        Map<String, Object> properties = new HashMap<>();
        properties.put("businessCode", ex.getBusinessCode());
        if (shouldIncludeStackTrace(request)) {
            properties.put("stackTrace", ExceptionUtils.getStackTrace(ex));
        }
        return problemDetailBuilder.build(
            HttpStatus.BAD_REQUEST,
            ErrorCodes.BUSINESS_ERROR,
            ex.getMessage(),
            properties
        );
    }
    
    // 处理资源未找到
    @ExceptionHandler(ResourceNotFoundException.class)
    ProblemDetail handleResourceNotFound(ResourceNotFoundException ex, WebRequest request) {
        Map<String, Object> properties = new HashMap<>();
        properties.put("resourceId", ex.getResourceId());
        if (shouldIncludeStackTrace(request)) {
            properties.put("stackTrace", ExceptionUtils.getStackTrace(ex));
        }
        return problemDetailBuilder.build(
            HttpStatus.NOT_FOUND,
            ErrorCodes.USER_NOT_FOUND,
            ex.getMessage(),
            properties
        );
    }
    
    // 处理访问拒绝异常
    @ExceptionHandler(AccessDeniedException.class)
    ProblemDetail handleAccessDenied(AccessDeniedException ex, WebRequest request) {
        Map<String, Object> properties = new HashMap<>();
        if (shouldIncludeStackTrace(request)) {
            properties.put("stackTrace", ExceptionUtils.getStackTrace(ex));
        }
        return problemDetailBuilder.build(
            HttpStatus.FORBIDDEN,
            ErrorCodes.ACCESS_DENIED,
            "Access denied",
            properties
        );
    }
    
    // 处理认证异常
    @ExceptionHandler(AuthenticationException.class)
    ProblemDetail handleAuthenticationException(AuthenticationException ex, WebRequest request) {
        Map<String, Object> properties = new HashMap<>();
        if (shouldIncludeStackTrace(request)) {
            properties.put("stackTrace", ExceptionUtils.getStackTrace(ex));
        }
        return problemDetailBuilder.build(
            HttpStatus.UNAUTHORIZED,
            ErrorCodes.AUTHENTICATION_FAILED,
            "Authentication failed",
            properties
        );
    }
    
    // 处理其他异常
    @ExceptionHandler(Exception.class)
    ProblemDetail handleOthers(Exception ex, WebRequest request) {
        Map<String, Object> properties = new HashMap<>();
        if (shouldIncludeStackTrace(request)) {
            properties.put("stackTrace", ExceptionUtils.getStackTrace(ex));
        }
        return problemDetailBuilder.build(
            HttpStatus.INTERNAL_SERVER_ERROR,
            ErrorCodes.UNKNOWN_ERROR,
            "An unexpected error occurred",
            properties
        );
    }
}
```

### 2.7 使用示例
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

// 基础异常类
@Getter
public abstract class BaseException extends RuntimeException {
    private final int errorCode;
    private final Map<String, Object> properties;
    
    protected BaseException(int errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.properties = new HashMap<>();
    }
    
    public void addProperty(String key, Object value) {
        properties.put(key, value);
    }
    
    public Map<String, Object> getProperties() {
        return Collections.unmodifiableMap(properties);
    }
}

// 业务异常
@Getter
public class BusinessException extends BaseException {
    private final String businessCode;
    
    public BusinessException(String message, String businessCode) {
        super(ErrorCodes.BUSINESS_ERROR, message);
        this.businessCode = businessCode;
    }
}

// 资源未找到异常
@Getter
public class ResourceNotFoundException extends BaseException {
    private final String resourceId;
    
    public ResourceNotFoundException(String message, String resourceId) {
        super(ErrorCodes.USER_NOT_FOUND, message);
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
        messageSource.setBasename("messages/errors");
        messageSource.setDefaultLocale(Locale.ENGLISH);
        messageSource.setDefaultEncoding("UTF-8");
        messageSource.setCacheSeconds(3600);
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
   
   实现示例见 2.6 节的 `GlobalExceptionHandler`

## 4. 最佳实践

### 4.1 方案选择
1. **项目规模与方案对应**
   - 小型项目：使用基础实现（1.3节）
   - 中型项目：使用完整功能实现（2.x节）
   - 大型项目：考虑高级特性（3.x节）

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
   - 继承 `BaseException`
   - 使用 `properties` 存储额外信息
   - 提供清晰的错误消息
   
   示例见 2.7 节的异常类定义

### 4.3 环境配置
1. **开发环境**
   ```yaml
   api:
     error:
       type-prefix: "https://api.example.com/problems"
       docs-url-template: "https://api.example.com/docs/errors/%s"
       include-stacktrace: ALWAYS
   ```

2. **生产环境**
   ```yaml
   api:
     error:
       type-prefix: "https://api.example.com/problems"
       docs-url-template: "https://api.example.com/docs/errors/%s"
       include-stacktrace: NEVER
   ```

### 4.4 安全考虑
1. **信息隐藏**
   - 生产环境中隐藏技术细节
   - 避免在错误消息中包含敏感信息
   - 使用适当的抽象级别描述错误
   - 生产环境禁用堆栈跟踪或仅对特定IP开放
   - 使用 ON_PARAM 模式时注意访问控制

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
    assertThat(problem.getProperty("resourceId")).isEqualTo("test-id");
    assertThat(problem.getType().toString())
        .isEqualTo("https://api.example.com/problems/2000");
}

@Test
void whenValidationFails_thenReturn400() {
    MethodArgumentNotValidException ex = // ... 
    ProblemDetail problem = handler.handleMethodArgumentNotValid(ex);
    
    assertThat(problem.getStatus()).isEqualTo(400);
    assertThat(problem.getProperty("code")).isEqualTo(1001);
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

### 4.8 设计决策说明

1. **错误信息分层**
   - 用户层：通过 `detail` 字段提供用户友好的错误信息
   - 开发者层：通过日志系统记录详细信息
   - 文档层：通过 `type` 和 `moreInfo` 提供错误处理指南

2. **属性控制**
   - 标准属性：严格遵循 RFC 7807 规范
   - 扩展属性：仅添加 `code`、`timestamp` 等必要信息
   - 敏感信息：通过日志系统记录，不在响应中返回

3. **URL 设计**
   - `type`：用于标识错误类型，如 "https://api.example.com/problems/user-not-found"
   - `moreInfo`：用于提供错误处理文档，如 "https://api.example.com/docs/errors/2000"
   - 分离关注点：错误类型和文档URL分开管理