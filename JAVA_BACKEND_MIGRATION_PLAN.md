# Runcode Java后端改造详细方案

## 目录
1. [项目概述](#1-项目概述)
2. [当前架构分析](#2-当前架构分析)
3. [Java技术栈选型](#3-java技术栈选型)
4. [详细架构设计](#4-详细架构设计)
5. [Docker集成方案](#5-docker集成方案)
6. [API接口设计](#6-api接口设计)
7. [核心功能实现](#7-核心功能实现)
8. [安全性设计](#8-安全性设计)
9. [性能优化方案](#9-性能优化方案)
10. [部署方案](#10-部署方案)
11. [迁移步骤](#11-迁移步骤)
12. [风险评估与应对](#12-风险评估与应对)

---

## 1. 项目概述

### 1.1 项目简介
Runcode 是一个在线代码运行编辑器，支持多种编程语言（C++, Java, Python, Go, Rust, Node.js, PHP, C#, TypeScript等）的在线编译和运行。

### 1.2 改造目标
将后端从 **Node.js + Koa + TypeScript** 迁移到 **Java 生态系统**，实现：
- 更好的性能和并发处理能力
- 更成熟的企业级应用支持
- 更强的类型安全和代码可维护性
- 更丰富的Docker操作库支持
- 更好的多线程和资源管理能力

---

## 2. 当前架构分析

### 2.1 技术栈
**前端：**
- React + TypeScript
- Vite 构建工具
- Monaco Editor 代码编辑器
- Mobx 状态管理
- TailwindCSS 样式

**后端：**
- Node.js (>= 16)
- Koa Web框架
- TypeScript
- routing-controllers (装饰器路由)
- dockerode (Docker客户端)
- log4js 日志

### 2.2 核心功能模块
```
server/src/
├── app.ts                 # 应用入口
├── controller/
│   └── code.ts           # 代码执行控制器
├── docker/
│   └── index.ts          # Docker操作封装
├── config/
│   └── docker.ts         # Docker配置
├── logger/
│   └── index.ts          # 日志系统
└── utils/
    ├── type.ts           # 类型定义
    └── helper.ts         # 工具函数
```

### 2.3 核心流程
```
用户提交代码
    ↓
Controller接收请求 (code, type, stdin, version)
    ↓
Docker模块处理
    ↓
创建容器 → 写入代码 → 执行代码 → 获取输出
    ↓
返回结果 (output, code, time, message)
```

### 2.4 支持的语言
- C++ (11.5, 14.2)
- C (使用 GCC)
- Java (8, 11, 17, 20)
- Python (2.7.18, 3.9.18)
- Node.js (16, 18, 20, 22)
- Go (1.18, 1.20, 1.23)
- Rust (1.83.0)
- PHP (7.4, 8.4)
- C# (.NET 6.12)
- TypeScript

---

## 3. Java技术栈选型

### 3.1 核心框架选择

#### 选项对比

| 框架 | 优点 | 缺点 | 推荐度 |
|------|------|------|--------|
| **Spring Boot** | 生态成熟、功能完善、社区活跃 | 相对重量级 | ⭐⭐⭐⭐⭐ |
| Spring WebFlux | 响应式编程、高并发 | 学习曲线陡峭 | ⭐⭐⭐⭐ |
| Micronaut | 启动快、内存占用小 | 生态相对较小 | ⭐⭐⭐ |
| Quarkus | 原生编译、云原生 | 社区较新 | ⭐⭐⭐ |

**推荐方案：Spring Boot 3.x**

理由：
- 成熟稳定的生态系统
- 丰富的Docker集成库
- 完善的监控和管理工具
- 强大的依赖注入和AOP支持
- 易于团队协作和维护

### 3.2 技术栈组成

```yaml
核心框架:
  - Spring Boot: 3.2.0+
  - Spring MVC: Web框架
  - Spring Validation: 参数校验

Docker集成:
  - docker-java: 3.3.x (官方推荐的Java Docker客户端)

数据处理:
  - Jackson: JSON序列化
  - Lombok: 减少样板代码

日志系统:
  - Logback: 日志实现
  - SLF4J: 日志门面

工具库:
  - Apache Commons Lang3: 工具类
  - Guava: Google工具库

构建工具:
  - Maven: 3.8+ (推荐) 或 Gradle 8+

监控与管理:
  - Spring Boot Actuator: 应用监控
  - Micrometer: 指标收集

测试框架:
  - JUnit 5: 单元测试
  - Mockito: Mock框架
  - TestContainers: Docker集成测试
```

### 3.3 JDK版本选择
- **推荐：JDK 17 (LTS)**
- 备选：JDK 21 (最新LTS)
- 原因：长期支持、性能优化、现代语言特性

---

## 4. 详细架构设计

### 4.1 整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                      前端 (React)                        │
│              Monaco Editor + API Client                  │
└────────────────────┬────────────────────────────────────┘
                     │ HTTP/REST API
                     ↓
┌─────────────────────────────────────────────────────────┐
│                   Spring Boot 后端                       │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Controller Layer                      │  │
│  │         (请求接收、参数校验、响应封装)              │  │
│  └──────────────────┬────────────────────────────────┘  │
│                     ↓                                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Service Layer                         │  │
│  │      (业务逻辑、代码执行编排、结果处理)            │  │
│  └──────────────────┬────────────────────────────────┘  │
│                     ↓                                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │            Docker Integration Layer                │  │
│  │    (容器管理、镜像选择、执行监控、资源控制)        │  │
│  └──────────────────┬────────────────────────────────┘  │
│                     ↓                                     │
│  ┌───────────────────────────────────────────────────┐  │
│  │           Infrastructure Layer                     │  │
│  │        (配置管理、日志、异常处理、监控)            │  │
│  └───────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────────────────────┘
                     │ Docker API
                     ↓
┌─────────────────────────────────────────────────────────┐
│                   Docker Engine                          │
│         (容器运行时、镜像管理、网络隔离)                │
└─────────────────────────────────────────────────────────┘
```

### 4.2 项目结构设计

```
runcode-java/
├── pom.xml                              # Maven配置
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── runcode/
│   │   │           ├── RunCodeApplication.java      # 应用启动类
│   │   │           │
│   │   │           ├── controller/                  # 控制器层
│   │   │           │   ├── CodeExecutionController.java
│   │   │           │   └── HealthController.java
│   │   │           │
│   │   │           ├── service/                     # 服务层
│   │   │           │   ├── CodeExecutionService.java
│   │   │           │   └── impl/
│   │   │           │       └── CodeExecutionServiceImpl.java
│   │   │           │
│   │   │           ├── docker/                      # Docker集成
│   │   │           │   ├── DockerClientManager.java
│   │   │           │   ├── DockerExecutor.java
│   │   │           │   ├── ContainerManager.java
│   │   │           │   └── config/
│   │   │           │       └── DockerImageConfig.java
│   │   │           │
│   │   │           ├── model/                       # 数据模型
│   │   │           │   ├── request/
│   │   │           │   │   └── CodeExecutionRequest.java
│   │   │           │   ├── response/
│   │   │           │   │   └── CodeExecutionResponse.java
│   │   │           │   ├── dto/
│   │   │           │   │   └── ExecutionResult.java
│   │   │           │   └── enums/
│   │   │           │       ├── CodeType.java
│   │   │           │       ├── ExecutionStatus.java
│   │   │           │       └── LanguageVersion.java
│   │   │           │
│   │   │           ├── config/                      # 配置类
│   │   │           │   ├── DockerConfig.java
│   │   │           │   ├── ExecutorConfig.java
│   │   │           │   ├── CorsConfig.java
│   │   │           │   └── LoggingConfig.java
│   │   │           │
│   │   │           ├── exception/                   # 异常处理
│   │   │           │   ├── GlobalExceptionHandler.java
│   │   │           │   ├── DockerExecutionException.java
│   │   │           │   ├── TimeoutException.java
│   │   │           │   └── InvalidRequestException.java
│   │   │           │
│   │   │           ├── util/                        # 工具类
│   │   │           │   ├── StringUtils.java
│   │   │           │   ├── OutputFormatter.java
│   │   │           │   └── CodeValidator.java
│   │   │           │
│   │   │           └── aspect/                      # AOP切面
│   │   │               ├── LoggingAspect.java
│   │   │               └── PerformanceAspect.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml                      # 主配置
│   │       ├── application-dev.yml                  # 开发环境
│   │       ├── application-prod.yml                 # 生产环境
│   │       ├── logback-spring.xml                   # 日志配置
│   │       └── docker-images.yml                    # Docker镜像配置
│   │
│   └── test/
│       └── java/
│           └── com/
│               └── runcode/
│                   ├── controller/                  # 控制器测试
│                   ├── service/                     # 服务测试
│                   └── docker/                      # Docker集成测试
│
├── docker/                                          # Docker镜像
│   ├── cpp/
│   ├── java/
│   ├── python/
│   └── ...
│
├── scripts/                                         # 脚本文件
│   ├── build-images.sh
│   └── deploy.sh
│
└── README.md
```

### 4.3 核心类设计

#### 4.3.1 Controller 层

```java
@RestController
@RequestMapping("/api/code")
@Validated
@Slf4j
public class CodeExecutionController {

    private final CodeExecutionService codeExecutionService;

    @PostMapping("/run")
    public ResponseEntity<CodeExecutionResponse> executeCode(
        @Valid @RequestBody CodeExecutionRequest request) {
        // 控制器实现
    }
}
```

#### 4.3.2 Service 层

```java
@Service
public class CodeExecutionServiceImpl implements CodeExecutionService {

    private final DockerExecutor dockerExecutor;
    private final OutputFormatter outputFormatter;

    @Override
    public ExecutionResult executeCode(CodeExecutionRequest request) {
        // 服务实现
    }
}
```

#### 4.3.3 Docker 集成层

```java
@Component
public class DockerExecutor {

    private final DockerClient dockerClient;
    private final ContainerManager containerManager;

    public ExecutionResult execute(CodeExecutionRequest request) {
        // Docker执行实现
    }
}
```

---

## 5. Docker集成方案

### 5.1 docker-java 库介绍

**docker-java** 是Java生态中最成熟的Docker客户端库，由Docker官方认可。

#### Maven依赖
```xml
<dependency>
    <groupId>com.github.docker-java</groupId>
    <artifactId>docker-java-core</artifactId>
    <version>3.3.6</version>
</dependency>
<dependency>
    <groupId>com.github.docker-java</groupId>
    <artifactId>docker-java-transport-httpclient5</artifactId>
    <version>3.3.6</version>
</dependency>
```

### 5.2 Docker客户端配置

```java
@Configuration
public class DockerConfig {

    @Value("${docker.host:unix:///var/run/docker.sock}")
    private String dockerHost;

    @Value("${docker.port:2375}")
    private Integer dockerPort;

    @Value("${docker.connection-type:socket}")
    private String connectionType;

    @Bean
    public DockerClient dockerClient() {
        DockerClientConfig config;

        if ("tcp".equalsIgnoreCase(connectionType)) {
            config = DefaultDockerClientConfig.createDefaultConfigBuilder()
                .withDockerHost("tcp://" + dockerHost + ":" + dockerPort)
                .withDockerTlsVerify(false)
                .build();
        } else {
            config = DefaultDockerClientConfig.createDefaultConfigBuilder()
                .withDockerHost("unix:///var/run/docker.sock")
                .build();
        }

        return DockerClientBuilder.getInstance(config)
            .withDockerHttpClient(
                new ApacheDockerHttpClient.Builder()
                    .dockerHost(config.getDockerHost())
                    .maxConnections(100)
                    .connectionTimeout(Duration.ofSeconds(30))
                    .responseTimeout(Duration.ofSeconds(45))
                    .build()
            )
            .build();
    }
}
```

### 5.3 容器执行流程

```java
@Component
@Slf4j
public class ContainerManager {

    private final DockerClient dockerClient;

    @Value("${docker.execution.timeout:6000}")
    private long executionTimeout;

    public ExecutionResult executeInContainer(
        String image,
        String[] command,
        String code,
        String stdin) {

        String containerId = null;
        try {
            // 1. 创建容器
            CreateContainerResponse container = dockerClient
                .createContainerCmd(image)
                .withCmd(command)
                .withTty(true)
                .withAttachStdout(true)
                .withAttachStderr(true)
                .withNetworkDisabled(true)  // 禁用网络
                .withHostConfig(
                    HostConfig.newHostConfig()
                        .withMemory(256 * 1024 * 1024L)  // 256MB内存限制
                        .withNanoCPUs(1000000000L)        // 1核CPU限制
                        .withPidsLimit(50L)               // 进程数限制
                )
                .exec();

            containerId = container.getId();

            // 2. 启动容器
            dockerClient.startContainerCmd(containerId).exec();

            // 3. 等待执行完成（带超时）
            WaitContainerResultCallback callback = new WaitContainerResultCallback();
            dockerClient.waitContainerCmd(containerId)
                .exec(callback);

            boolean completed = callback.awaitCompletion(
                executionTimeout,
                TimeUnit.MILLISECONDS
            );

            // 4. 获取输出
            String output = getContainerOutput(containerId);

            // 5. 检查是否超时
            if (!completed) {
                return ExecutionResult.timeout("执行超时");
            }

            // 6. 返回结果
            return ExecutionResult.success(output);

        } catch (Exception e) {
            log.error("容器执行失败", e);
            return ExecutionResult.error(e.getMessage());
        } finally {
            // 7. 清理容器
            if (containerId != null) {
                removeContainer(containerId);
            }
        }
    }

    private String getContainerOutput(String containerId) {
        try (ByteArrayOutputStream stdout = new ByteArrayOutputStream();
             ByteArrayOutputStream stderr = new ByteArrayOutputStream()) {

            dockerClient.logContainerCmd(containerId)
                .withStdOut(true)
                .withStdErr(true)
                .withFollowStream(true)
                .exec(new LogContainerResultCallback() {
                    @Override
                    public void onNext(Frame frame) {
                        try {
                            if (frame.getStreamType() == StreamType.STDOUT) {
                                stdout.write(frame.getPayload());
                            } else if (frame.getStreamType() == StreamType.STDERR) {
                                stderr.write(frame.getPayload());
                            }
                        } catch (IOException e) {
                            log.error("写入输出失败", e);
                        }
                    }
                })
                .awaitCompletion(5, TimeUnit.SECONDS);

            String stdoutStr = stdout.toString(StandardCharsets.UTF_8);
            String stderrStr = stderr.toString(StandardCharsets.UTF_8);

            return StringUtils.isNotBlank(stderrStr) ? stderrStr : stdoutStr;

        } catch (Exception e) {
            log.error("获取容器输出失败", e);
            return "获取输出失败: " + e.getMessage();
        }
    }

    private void removeContainer(String containerId) {
        try {
            dockerClient.removeContainerCmd(containerId)
                .withForce(true)
                .exec();
        } catch (Exception e) {
            log.error("删除容器失败: {}", containerId, e);
        }
    }
}
```

### 5.4 镜像配置管理

```java
@Component
@ConfigurationProperties(prefix = "docker.images")
@Data
public class DockerImageConfig {

    private Map<String, LanguageConfig> languages = new HashMap<>();

    @Data
    public static class LanguageConfig {
        private String defaultVersion;
        private Map<String, ImageInfo> versions;
    }

    @Data
    public static class ImageInfo {
        private String image;
        private String compileCommand;
        private String executeCommand;
        private String fileSuffix;
        private String filePrefix = "code";
    }

    public ImageInfo getImageInfo(CodeType codeType, String version) {
        LanguageConfig config = languages.get(codeType.name().toLowerCase());
        if (config == null) {
            throw new IllegalArgumentException("不支持的语言: " + codeType);
        }

        String ver = version != null ? version : config.getDefaultVersion();
        ImageInfo info = config.getVersions().get(ver);

        if (info == null) {
            throw new IllegalArgumentException(
                String.format("不支持的版本: %s %s", codeType, ver)
            );
        }

        return info;
    }
}
```

**docker-images.yml 配置示例：**

```yaml
docker:
  images:
    languages:
      cpp:
        default-version: "14.2"
        versions:
          "11.5":
            image: "cpp:11.5"
            compile-command: "g++ code.cpp -o code.out"
            execute-command: "./code.out"
            file-suffix: "cpp"
          "14.2":
            image: "cpp:14.2"
            compile-command: "g++ code.cpp -o code.out"
            execute-command: "./code.out"
            file-suffix: "cpp"

      java:
        default-version: "17"
        versions:
          "8":
            image: "java:8"
            compile-command: "javac Code.java"
            execute-command: "java Code"
            file-suffix: "java"
            file-prefix: "Code"
          "11":
            image: "java:11"
            compile-command: "javac Code.java"
            execute-command: "java Code"
            file-suffix: "java"
            file-prefix: "Code"
          "17":
            image: "java:17"
            compile-command: "javac Code.java"
            execute-command: "java Code"
            file-suffix: "java"
            file-prefix: "Code"

      python:
        default-version: "3.9.18"
        versions:
          "2.7.18":
            image: "python:2.7.18"
            execute-command: "python code.py"
            file-suffix: "py"
          "3.9.18":
            image: "python:3.9.18"
            execute-command: "python3 code.py"
            file-suffix: "py"
```

---

## 6. API接口设计

### 6.1 请求模型

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CodeExecutionRequest {

    @NotBlank(message = "代码不能为空")
    @Size(max = 500000, message = "代码长度不能超过500000字符")
    private String code;

    @NotNull(message = "语言类型不能为空")
    private CodeType type;

    @Size(max = 500000, message = "输入数据长度不能超过500000字符")
    private String stdin;

    private String version;
}
```

```java
public enum CodeType {
    CPP("cpp", "C++"),
    C("c", "C"),
    JAVA("java", "Java"),
    PYTHON("python", "Python"),
    NODEJS("nodejs", "Node.js"),
    GO("go", "Go"),
    RUST("rust", "Rust"),
    PHP("php", "PHP"),
    DOTNET("dotnet", "C#"),
    TYPESCRIPT("typescript", "TypeScript");

    private final String code;
    private final String displayName;

    // getter, setter, constructor
}
```

### 6.2 响应模型

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CodeExecutionResponse {

    private Integer code;  // 0: 成功, 1: 失败

    private String message;

    private ExecutionData data;

    @Data
    @Builder
    public static class ExecutionData {
        private String output;
        private ExecutionStatus status;
        private Long executionTime;  // 毫秒
        private String error;
    }

    public static CodeExecutionResponse success(ExecutionData data) {
        return CodeExecutionResponse.builder()
            .code(0)
            .data(data)
            .build();
    }

    public static CodeExecutionResponse error(String message) {
        return CodeExecutionResponse.builder()
            .code(1)
            .message(message)
            .build();
    }
}
```

```java
public enum ExecutionStatus {
    SUCCESS,    // 执行成功
    TIMEOUT,    // 超时
    ERROR,      // 运行错误
    COMPILE_ERROR  // 编译错误
}
```

### 6.3 API端点

```
POST /api/code/run
Content-Type: application/json

Request Body:
{
  "code": "print('Hello World')",
  "type": "PYTHON",
  "stdin": "",
  "version": "3.9.18"
}

Response:
{
  "code": 0,
  "message": null,
  "data": {
    "output": "Hello World\n",
    "status": "SUCCESS",
    "executionTime": 1234,
    "error": null
  }
}
```

---

## 7. 核心功能实现

### 7.1 代码执行服务

```java
@Service
@Slf4j
public class CodeExecutionServiceImpl implements CodeExecutionService {

    private final ContainerManager containerManager;
    private final DockerImageConfig imageConfig;
    private final OutputFormatter outputFormatter;
    private final ExecutorService executorService;

    public CodeExecutionServiceImpl(
        ContainerManager containerManager,
        DockerImageConfig imageConfig,
        OutputFormatter outputFormatter,
        @Qualifier("codeExecutor") ExecutorService executorService) {
        this.containerManager = containerManager;
        this.imageConfig = imageConfig;
        this.outputFormatter = outputFormatter;
        this.executorService = executorService;
    }

    @Override
    public ExecutionResult executeCode(CodeExecutionRequest request) {
        long startTime = System.currentTimeMillis();

        try {
            // 1. 获取镜像配置
            DockerImageConfig.ImageInfo imageInfo =
                imageConfig.getImageInfo(request.getType(), request.getVersion());

            // 2. 构建执行命令
            String[] command = buildCommand(
                imageInfo,
                request.getCode(),
                request.getStdin()
            );

            // 3. 异步执行（防止阻塞）
            Future<ExecutionResult> future = executorService.submit(
                () -> containerManager.executeInContainer(
                    imageInfo.getImage(),
                    command,
                    request.getCode(),
                    request.getStdin()
                )
            );

            // 4. 等待结果（带超时）
            ExecutionResult result = future.get(10, TimeUnit.SECONDS);

            // 5. 格式化输出
            String formattedOutput = outputFormatter.format(result.getOutput());
            result.setOutput(formattedOutput);

            // 6. 记录执行时间
            long executionTime = System.currentTimeMillis() - startTime;
            result.setExecutionTime(executionTime);

            return result;

        } catch (TimeoutException e) {
            log.error("代码执行超时", e);
            return ExecutionResult.timeout("执行超时");
        } catch (Exception e) {
            log.error("代码执行失败", e);
            return ExecutionResult.error("执行失败: " + e.getMessage());
        }
    }

    private String[] buildCommand(
        DockerImageConfig.ImageInfo imageInfo,
        String code,
        String stdin) {

        StringBuilder script = new StringBuilder();

        // 写入代码文件
        String fileName = imageInfo.getFilePrefix() + "." + imageInfo.getFileSuffix();
        script.append("cat > ").append(fileName).append(" << 'EOF'\n");
        script.append(code).append("\n");
        script.append("EOF\n");

        // 写入输入文件（如果有）
        if (StringUtils.isNotBlank(stdin)) {
            script.append("cat > input.txt << 'EOF'\n");
            script.append(stdin).append("\n");
            script.append("EOF\n");
        }

        // 编译（如果需要）
        if (StringUtils.isNotBlank(imageInfo.getCompileCommand())) {
            script.append(imageInfo.getCompileCommand()).append("\n");
        }

        // 执行
        script.append(imageInfo.getExecuteCommand());
        if (StringUtils.isNotBlank(stdin)) {
            script.append(" < input.txt");
        }

        return new String[]{"bash", "-c", script.toString()};
    }
}
```

### 7.2 输出格式化

```java
@Component
public class OutputFormatter {

    private static final int MAX_OUTPUT_LENGTH = 8200;
    private static final int MAX_LINES = 200;

    public String format(String output) {
        if (output == null) {
            return "";
        }

        // 1. 长度限制
        if (output.length() > MAX_OUTPUT_LENGTH) {
            output = output.substring(0, 4000) +
                     "\n...[输出过长，已截断]...\n" +
                     output.substring(output.length() - 4000);
        }

        // 2. 行数限制
        String[] lines = output.split("\n");
        if (lines.length > MAX_LINES) {
            List<String> result = new ArrayList<>();

            // 前100行
            for (int i = 0; i < 100; i++) {
                result.add(lines[i]);
            }

            result.add("...[输出行数过多，已折叠]...");

            // 后100行
            for (int i = lines.length - 100; i < lines.length; i++) {
                result.add(lines[i]);
            }

            output = String.join("\n", result);
        }

        return output;
    }
}
```

### 7.3 参数校验

```java
@Component
public class CodeValidator {

    private static final int MAX_CODE_LENGTH = 500000;
    private static final int MAX_STDIN_LENGTH = 500000;

    public void validate(CodeExecutionRequest request) {
        // 1. 代码长度校验
        if (request.getCode() == null || request.getCode().isEmpty()) {
            throw new InvalidRequestException("代码不能为空");
        }

        if (request.getCode().length() > MAX_CODE_LENGTH) {
            throw new InvalidRequestException("代码长度超过限制");
        }

        // 2. 输入长度校验
        if (request.getStdin() != null &&
            request.getStdin().length() > MAX_STDIN_LENGTH) {
            throw new InvalidRequestException("输入数据长度超过限制");
        }

        // 3. 代码类型校验
        if (request.getType() == null) {
            throw new InvalidRequestException("语言类型不能为空");
        }

        // 4. 危险代码检测（基础）
        checkDangerousCode(request.getCode());
    }

    private void checkDangerousCode(String code) {
        // 检测一些明显的危险操作
        List<String> dangerousPatterns = Arrays.asList(
            "rm -rf", "format", "del /f",
            "DROP TABLE", "DROP DATABASE"
        );

        for (String pattern : dangerousPatterns) {
            if (code.contains(pattern)) {
                log.warn("检测到潜在危险代码: {}", pattern);
            }
        }
    }
}
```

---

## 8. 安全性设计

### 8.1 容器安全隔离

```java
@Component
public class SecurityConfig {

    public HostConfig createSecureHostConfig() {
        return HostConfig.newHostConfig()
            // 内存限制：256MB
            .withMemory(256 * 1024 * 1024L)
            .withMemorySwap(256 * 1024 * 1024L)

            // CPU限制：1核
            .withNanoCPUs(1000000000L)
            .withCpuQuota(100000L)
            .withCpuPeriod(100000L)

            // 进程数限制
            .withPidsLimit(50L)

            // 禁用特权模式
            .withPrivileged(false)

            // 只读根文件系统（可选）
            // .withReadonlyRootfs(true)

            // 禁用新权限
            .withSecurityOpts(Arrays.asList("no-new-privileges"))

            // 限制设备访问
            .withCapDrop(Arrays.asList("ALL"))

            // 磁盘IO限制
            .withBlkioWeight(500);
    }
}
```

### 8.2 网络隔离

```java
// 创建容器时禁用网络
createContainerCmd.withNetworkDisabled(true);

// 或者使用自定义网络
createContainerCmd.withHostConfig(
    HostConfig.newHostConfig()
        .withNetworkMode("none")  // 完全隔离
);
```

### 8.3 输入验证与过滤

```java
@Component
public class InputSanitizer {

    public String sanitize(String input) {
        if (input == null) {
            return "";
        }

        // 移除潜在的危险字符
        // 根据需求调整
        return input
            .replaceAll("[\\x00-\\x08\\x0B-\\x0C\\x0E-\\x1F]", "")
            .trim();
    }

    public boolean isSafe(String code) {
        // 实现更复杂的安全检查
        // 例如：检测shell注入、SQL注入等
        return true;
    }
}
```

### 8.4 超时控制

```java
@Configuration
public class ExecutorConfig {

    @Bean(name = "codeExecutor")
    public ExecutorService codeExecutor() {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            10,  // 核心线程数
            50,  // 最大线程数
            60L, TimeUnit.SECONDS,  // 空闲超时
            new LinkedBlockingQueue<>(100),  // 队列大小
            new ThreadFactoryBuilder()
                .setNameFormat("code-executor-%d")
                .build(),
            new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略
        );

        return executor;
    }
}
```

### 8.5 Rate Limiting (限流)

```java
@Component
public class RateLimiter {

    private final Cache<String, AtomicInteger> requestCache;

    public RateLimiter() {
        this.requestCache = CacheBuilder.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .build();
    }

    public boolean allowRequest(String clientIp) {
        AtomicInteger count = requestCache.getIfPresent(clientIp);

        if (count == null) {
            requestCache.put(clientIp, new AtomicInteger(1));
            return true;
        }

        // 每分钟最多60次请求
        return count.incrementAndGet() <= 60;
    }
}
```

---

## 9. 性能优化方案

### 9.1 线程池配置

```yaml
# application.yml
executor:
  core-pool-size: 10
  max-pool-size: 50
  queue-capacity: 100
  keep-alive-seconds: 60
  thread-name-prefix: "code-exec-"
```

### 9.2 容器复用池（可选优化）

```java
@Component
public class ContainerPool {

    private final Map<String, Queue<String>> containerPools = new ConcurrentHashMap<>();
    private final DockerClient dockerClient;

    @Value("${docker.pool.max-size:10}")
    private int maxPoolSize;

    /**
     * 获取或创建容器
     */
    public String acquireContainer(String image) {
        Queue<String> pool = containerPools.computeIfAbsent(
            image,
            k -> new ConcurrentLinkedQueue<>()
        );

        String containerId = pool.poll();

        if (containerId == null) {
            // 创建新容器
            containerId = createContainer(image);
        }

        return containerId;
    }

    /**
     * 归还容器
     */
    public void releaseContainer(String image, String containerId) {
        Queue<String> pool = containerPools.get(image);

        if (pool != null && pool.size() < maxPoolSize) {
            // 清理容器状态
            cleanContainer(containerId);
            pool.offer(containerId);
        } else {
            // 删除容器
            removeContainer(containerId);
        }
    }

    private void cleanContainer(String containerId) {
        // 停止容器
        // 清理文件
        // 重置状态
    }
}
```

### 9.3 异步日志

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <appender-ref ref="FILE" />
    </appender>

    <root level="INFO">
        <appender-ref ref="ASYNC" />
    </root>
</configuration>
```

### 9.4 响应压缩

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,text/html,text/plain
    min-response-size: 1024
```

### 9.5 缓存策略（镜像信息）

```java
@Service
public class ImageCacheService {

    private final LoadingCache<String, DockerImageConfig.ImageInfo> imageCache;

    public ImageCacheService(DockerImageConfig config) {
        this.imageCache = CacheBuilder.newBuilder()
            .maximumSize(100)
            .expireAfterWrite(1, TimeUnit.HOURS)
            .build(new CacheLoader<String, DockerImageConfig.ImageInfo>() {
                @Override
                public DockerImageConfig.ImageInfo load(String key) {
                    String[] parts = key.split(":");
                    return config.getImageInfo(
                        CodeType.valueOf(parts[0]),
                        parts[1]
                    );
                }
            });
    }

    public DockerImageConfig.ImageInfo getImageInfo(CodeType type, String version) {
        try {
            return imageCache.get(type + ":" + version);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

## 10. 部署方案

### 10.1 构建配置

#### Maven 配置 (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.runcode</groupId>
    <artifactId>runcode-java</artifactId>
    <version>1.0.0</version>
    <name>RunCode Java Backend</name>

    <properties>
        <java.version>17</java.version>
        <docker-java.version>3.3.6</docker-java.version>
    </properties>

    <dependencies>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Docker Java -->
        <dependency>
            <groupId>com.github.docker-java</groupId>
            <artifactId>docker-java-core</artifactId>
            <version>${docker-java.version}</version>
        </dependency>

        <dependency>
            <groupId>com.github.docker-java</groupId>
            <artifactId>docker-java-transport-httpclient5</artifactId>
            <version>${docker-java.version}</version>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Guava -->
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>32.1.3-jre</version>
        </dependency>

        <!-- Commons Lang3 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 10.2 配置文件

#### application.yml

```yaml
server:
  port: 39005
  compression:
    enabled: true
    mime-types: application/json,text/html,text/plain
  tomcat:
    threads:
      max: 200
      min-spare: 10

spring:
  application:
    name: runcode-java
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

# Docker配置
docker:
  connection-type: socket  # socket 或 tcp
  host: 127.0.0.1
  port: 2375
  execution:
    timeout: 6000  # 毫秒
  pool:
    enabled: false
    max-size: 10

# 执行器配置
executor:
  core-pool-size: 10
  max-pool-size: 50
  queue-capacity: 100
  keep-alive-seconds: 60

# 日志配置
logging:
  level:
    com.runcode: INFO
    com.github.dockerjava: WARN
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/runcode.log
    max-size: 100MB
    max-history: 30

# 监控配置
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

#### application-prod.yml

```yaml
docker:
  connection-type: tcp
  host: ${DOCKER_HOST:127.0.0.1}
  port: ${DOCKER_PORT:2375}

logging:
  level:
    com.runcode: INFO
    root: WARN
```

### 10.3 Dockerfile（可选，用于容器化部署）

```dockerfile
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY target/runcode-java-1.0.0.jar app.jar

EXPOSE 39005

ENV JAVA_OPTS="-Xms256m -Xmx512m"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### 10.4 部署脚本

#### deploy.sh

```bash
#!/bin/bash

echo "开始部署 RunCode Java 后端..."

# 1. 编译项目
echo "编译项目..."
mvn clean package -DskipTests

# 2. 停止旧服务
echo "停止旧服务..."
pid=$(ps -ef | grep "runcode-java" | grep -v grep | awk '{print $2}')
if [ -n "$pid" ]; then
    kill -9 $pid
    echo "已停止旧服务 (PID: $pid)"
fi

# 3. 启动新服务
echo "启动新服务..."
nohup java -jar target/runcode-java-1.0.0.jar \
    --spring.profiles.active=prod \
    > logs/runcode.out 2>&1 &

echo "服务已启动"
echo "查看日志: tail -f logs/runcode.out"
```

### 10.5 系统要求

**硬件要求：**
- CPU: 4核以上
- 内存: 8GB以上
- 磁盘: 50GB以上

**软件要求：**
- JDK 17+
- Docker Engine 20.10+
- Maven 3.8+ (开发环境)

---

## 11. 迁移步骤

### 11.1 准备阶段（1-2天）

1. **环境准备**
   - 安装JDK 17
   - 安装Maven 3.8+
   - 确认Docker环境正常

2. **项目初始化**
   ```bash
   # 创建项目目录
   mkdir runcode-java
   cd runcode-java

   # 初始化Maven项目
   mvn archetype:generate \
       -DgroupId=com.runcode \
       -DartifactId=runcode-java \
       -DarchetypeArtifactId=maven-archetype-quickstart \
       -DinteractiveMode=false

   # 配置pom.xml
   # 添加Spring Boot和docker-java依赖
   ```

3. **代码结构搭建**
   - 创建包结构
   - 配置Spring Boot
   - 配置Docker客户端

### 11.2 开发阶段（7-10天）

**第1-2天：基础框架**
- [ ] 搭建Spring Boot项目
- [ ] 配置Docker客户端
- [ ] 实现健康检查接口
- [ ] 配置日志系统

**第3-4天：Docker集成**
- [ ] 实现ContainerManager
- [ ] 实现镜像配置管理
- [ ] 实现容器创建和执行
- [ ] 实现容器清理机制

**第5-6天：业务逻辑**
- [ ] 实现CodeExecutionService
- [ ] 实现命令构建逻辑
- [ ] 实现输出格式化
- [ ] 实现参数校验

**第7-8天：控制器和API**
- [ ] 实现CodeExecutionController
- [ ] 实现全局异常处理
- [ ] 实现CORS配置
- [ ] API文档编写

**第9-10天：测试和优化**
- [ ] 单元测试
- [ ] 集成测试
- [ ] 性能测试
- [ ] 代码优化

### 11.3 测试阶段（3-5天）

1. **功能测试**
   - 测试所有支持的语言
   - 测试不同版本的运行环境
   - 测试输入输出功能
   - 测试超时机制

2. **性能测试**
   - 并发测试（JMeter/Gatling）
   - 负载测试
   - 压力测试
   - 资源占用测试

3. **安全测试**
   - 容器隔离测试
   - 资源限制测试
   - 恶意代码防护测试
   - API安全测试

### 11.4 部署阶段（2-3天）

1. **预生产环境**
   - 部署到测试服务器
   - 验证Docker连接
   - 验证所有镜像可用
   - 压力测试

2. **生产环境准备**
   - 备份当前Node.js服务
   - 准备回滚方案
   - 更新监控配置
   - 通知相关人员

3. **正式部署**
   - 停止Node.js服务
   - 启动Java服务
   - 验证服务正常
   - 监控日志和指标

4. **灰度发布（推荐）**
   - 10%流量 → Java服务
   - 观察24小时
   - 50%流量 → Java服务
   - 观察24小时
   - 100%流量 → Java服务

### 11.5 验收阶段（1-2天）

- [ ] 功能验收
- [ ] 性能验收
- [ ] 安全验收
- [ ] 文档完善
- [ ] 知识转移

---

## 12. 风险评估与应对

### 12.1 技术风险

| 风险项 | 可能性 | 影响 | 应对措施 |
|--------|--------|------|----------|
| Docker库兼容性问题 | 中 | 高 | 提前充分测试docker-java库 |
| 性能不达标 | 低 | 高 | 进行基准测试，优化线程池配置 |
| 内存泄漏 | 中 | 高 | 代码审查，压力测试，监控内存 |
| 容器清理失败 | 中 | 中 | 实现定时清理机制 |

### 12.2 业务风险

| 风险项 | 可能性 | 影响 | 应对措施 |
|--------|--------|------|----------|
| 服务中断 | 低 | 高 | 灰度发布，准备回滚方案 |
| 功能遗漏 | 中 | 中 | 详细的功能对比清单 |
| API不兼容 | 低 | 高 | 保持API接口一致性 |

### 12.3 进度风险

| 风险项 | 可能性 | 影响 | 应对措施 |
|--------|--------|------|----------|
| 开发延期 | 中 | 中 | 预留缓冲时间，分阶段交付 |
| 测试不充分 | 中 | 高 | 自动化测试，增加测试人力 |
| 文档不完善 | 高 | 低 | 开发过程中同步编写文档 |

### 12.4 回滚方案

如果Java版本出现严重问题，需要立即回滚：

```bash
# 1. 停止Java服务
kill -9 $(ps -ef | grep runcode-java | awk '{print $2}')

# 2. 启动Node.js备份服务
cd server
pm2 start dist/app.js --name runcode

# 3. 验证服务
curl http://localhost:39005/health

# 4. 通知相关人员
```

---

## 13. 附录

### 13.1 完整的Maven依赖清单

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Boot Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Spring Boot Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Spring Boot AOP -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>

    <!-- Docker Java Core -->
    <dependency>
        <groupId>com.github.docker-java</groupId>
        <artifactId>docker-java-core</artifactId>
        <version>3.3.6</version>
    </dependency>

    <!-- Docker Java Transport -->
    <dependency>
        <groupId>com.github.docker-java</groupId>
        <artifactId>docker-java-transport-httpclient5</artifactId>
        <version>3.3.6</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- Guava -->
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>32.1.3-jre</version>
    </dependency>

    <!-- Commons Lang3 -->
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
    </dependency>

    <!-- Jackson -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>

    <!-- Micrometer Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Test Dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 13.2 性能对比（预期）

| 指标 | Node.js版本 | Java版本 | 提升 |
|------|-------------|----------|------|
| 单次执行响应时间 | 200ms | 180ms | 10% |
| 并发100 QPS | 95% < 500ms | 95% < 400ms | 20% |
| 内存占用 | 150MB | 200MB | -25% |
| CPU占用 | 15% | 12% | 20% |
| 容器清理成功率 | 98% | 99.5% | 1.5% |

### 13.3 参考资源

**Docker Java:**
- 官方文档: https://github.com/docker-java/docker-java
- API文档: https://docker-java.readthedocs.io/

**Spring Boot:**
- 官方文档: https://spring.io/projects/spring-boot
- 最佳实践: https://docs.spring.io/spring-boot/docs/current/reference/html/

**Docker:**
- Docker API: https://docs.docker.com/engine/api/
- Docker安全: https://docs.docker.com/engine/security/

---

## 总结

本方案详细描述了将Runcode项目从Node.js迁移到Java的完整流程，包括：

1. **技术选型**：Spring Boot + docker-java的成熟组合
2. **架构设计**：清晰的分层架构，易于维护和扩展
3. **安全保障**：容器隔离、资源限制、输入验证等多重保护
4. **性能优化**：线程池、容器复用、异步处理等优化手段
5. **部署方案**：完整的构建、部署、监控方案
6. **风险控制**：详细的风险评估和回滚方案

**预期收益：**
- ✅ 更好的性能和并发能力
- ✅ 更强的类型安全
- ✅ 更成熟的企业级支持
- ✅ 更好的监控和运维能力
- ✅ 更丰富的生态系统

**建议实施周期：** 2-3周（包含测试和部署）

**关键成功因素：**
1. 充分的测试（单元测试、集成测试、性能测试）
2. 灰度发布策略
3. 完善的监控和告警
4. 详细的文档和知识转移
5. 准备好的回滚方案

---

**文档版本：** 1.0
**创建日期：** 2025-11-06
**作者：** Claude
**审核状态：** 待审核
