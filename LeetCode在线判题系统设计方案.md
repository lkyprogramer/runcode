# LeetCode 在线判题系统后台设计方案（Java版）

## 目录

- [1. 系统概述](#1-系统概述)
- [2. 技术架构](#2-技术架构)
- [3. 技术栈选型](#3-技术栈选型)
- [4. 系统架构设计](#4-系统架构设计)
- [5. 数据库设计](#5-数据库设计)
- [6. 核心模块设计](#6-核心模块设计)
- [7. 代码执行引擎](#7-代码执行引擎)
- [8. 高并发解决方案](#8-高并发解决方案)
- [9. 性能监控与资源管理](#9-性能监控与资源管理)
- [10. 安全机制](#10-安全机制)
- [11. API 设计](#11-api-设计)
- [12. 部署方案](#12-部署方案)
- [13. 扩展性与优化](#13-扩展性与优化)
- [14. 参考现有项目的改进点](#14-参考现有项目的改进点)

---

## 1. 系统概述

### 1.1 系统定位

一个支持多编程语言的在线代码评测系统，类似 LeetCode，提供：
- 题目管理与展示
- 多语言代码提交与评测
- 实时测试用例执行
- 运行时间、内存占用监控
- 提交历史与统计分析
- 高并发代码执行调度

### 1.2 核心功能

| 功能模块 | 描述 |
|---------|------|
| **用户管理** | 用户注册、登录、权限管理 |
| **题目管理** | 题目增删改查、分类、难度标记 |
| **代码提交** | 支持多语言代码提交 |
| **判题引擎** | 代码编译、运行、测试用例校验 |
| **性能监控** | 运行时间、内存使用、CPU 占用 |
| **排行榜** | 基于执行时间和内存的排名 |
| **提交历史** | 个人提交记录、代码查看 |

### 1.3 非功能性需求

- **高并发**: 支持 1000+ QPS 代码提交
- **安全性**: 代码沙箱隔离，防止恶意代码
- **可靠性**: 99.9% 可用性
- **可扩展**: 支持水平扩展
- **低延迟**: 90% 请求在 5 秒内返回结果

---

## 2. 技术架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          用户层 (Client)                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  Web 浏览器   │  │  移动端 App   │  │   IDE 插件   │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
                                │
                         HTTPS / REST API
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       网关层 (Gateway Layer)                          │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  Spring Cloud Gateway / Nginx                             │       │
│  │  • 路由转发  • 负载均衡  • 限流  • 认证鉴权               │       │
│  └──────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
                ▼               ▼               ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  用户服务         │  │  题目服务         │  │  判题服务         │
│  (User Service)  │  │(Question Service)│  │ (Judge Service)  │
│                  │  │                  │  │                  │
│ • 注册登录       │  │ • 题目管理       │  │ • 代码提交       │
│ • 用户信息       │  │ • 测试用例       │  │ • 异步判题       │
│ • 权限管理       │  │ • 题解讨论       │  │ • 结果返回       │
└──────────────────┘  └──────────────────┘  └──────────────────┘
        │                      │                      │
        └──────────────────────┼──────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      基础设施层 (Infrastructure)                      │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │   MySQL     │ │    Redis    │ │  RabbitMQ   │ │    Nacos    │   │
│ │  (主数据库)  │ │   (缓存)    │ │  (消息队列) │ │ (注册中心)  │   │
│ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │   Docker    │ │    MinIO    │ │Elasticsearch│ │  Prometheus │   │
│ │(代码沙箱)    │ │ (对象存储)  │ │  (日志搜索) │ │   (监控)    │   │
│ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      判题沙箱集群 (Judge Sandbox)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Docker 容器 1 │  │ Docker 容器 2 │  │ Docker 容器 N │              │
│  │ • 代码编译    │  │ • 代码编译    │  │ • 代码编译    │              │
│  │ • 测试执行    │  │ • 测试执行    │  │ • 测试执行    │              │
│  │ • 资源隔离    │  │ • 资源隔离    │  │ • 资源隔离    │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 系统分层

```
┌─────────────────────────────────────────────┐
│          接入层 (Gateway Layer)              │
│  • API Gateway                               │
│  • Load Balancer                             │
│  • Rate Limiter                              │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│          应用层 (Application Layer)          │
│  • Controller (接收请求)                     │
│  • Service (业务逻辑)                        │
│  • DTO/VO (数据传输对象)                     │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│          领域层 (Domain Layer)               │
│  • Entity (领域实体)                         │
│  • Repository Interface (仓储接口)          │
│  • Domain Service (领域服务)                │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│          基础层 (Infrastructure Layer)       │
│  • Repository Implementation (仓储实现)     │
│  • External API Client (外部 API 客户端)    │
│  • Message Queue (消息队列)                 │
│  • Cache (缓存)                             │
└─────────────────────────────────────────────┘
```

---

## 3. 技术栈选型

### 3.1 后端技术栈

| 层级 | 技术选型 | 版本 | 说明 |
|-----|---------|------|------|
| **核心框架** | Spring Boot | 3.2.x | 微服务基础框架 |
| **微服务框架** | Spring Cloud Alibaba | 2023.x | 微服务生态 |
| **服务注册** | Nacos | 2.3.x | 服务发现与配置中心 |
| **API 网关** | Spring Cloud Gateway | 4.1.x | 统一网关 |
| **负载均衡** | Spring Cloud LoadBalancer | 4.1.x | 客户端负载均衡 |
| **数据库** | MySQL | 8.0+ | 主数据库 |
| **ORM 框架** | MyBatis-Plus | 3.5.x | 持久层框架 |
| **缓存** | Redis | 7.0+ | 分布式缓存 |
| **消息队列** | RabbitMQ | 3.12+ | 异步任务队列 |
| **容器化** | Docker | 24.0+ | 代码沙箱 |
| **Docker API** | docker-java | 3.3.x | Java Docker 客户端 |
| **对象存储** | MinIO | RELEASE.2024 | 代码文件存储 |
| **全文搜索** | Elasticsearch | 8.x | 题目搜索 |
| **分布式锁** | Redisson | 3.27.x | 分布式锁 |
| **限流熔断** | Sentinel | 1.8.x | 流量控制 |
| **认证授权** | Spring Security + JWT | 6.2.x | 安全框架 |
| **监控** | Prometheus + Grafana | - | 性能监控 |
| **链路追踪** | SkyWalking | 9.x | 分布式追踪 |
| **日志收集** | ELK (Elasticsearch + Logstash + Kibana) | 8.x | 日志分析 |
| **API 文档** | Knife4j (Swagger) | 4.4.x | 接口文档 |

### 3.2 开发工具与辅助

| 类别 | 工具 | 说明 |
|-----|------|------|
| **构建工具** | Maven 3.9+ | 项目管理 |
| **JDK 版本** | OpenJDK 17+ | Java 运行时 |
| **代码规范** | Checkstyle + SpotBugs | 代码质量检查 |
| **测试框架** | JUnit 5 + Mockito | 单元测试 |
| **压测工具** | JMeter / Gatling | 性能测试 |

---

## 4. 系统架构设计

### 4.1 微服务划分

#### 4.1.1 服务列表

```
oj-system/
├── oj-gateway/              # API 网关服务 (端口: 8080)
├── oj-user-service/         # 用户服务 (端口: 8081)
├── oj-question-service/     # 题目服务 (端口: 8082)
├── oj-judge-service/        # 判题服务 (端口: 8083)
├── oj-submission-service/   # 提交记录服务 (端口: 8084)
├── oj-ranking-service/      # 排行榜服务 (端口: 8085)
├── oj-common/               # 公共模块 (工具类、常量、异常)
├── oj-model/                # 数据模型模块 (Entity、DTO、VO)
└── oj-sdk/                  # SDK 模块 (对外接口)
```

#### 4.1.2 服务职责详解

**1. oj-gateway (API 网关)**
```java
功能：
├── 统一入口：所有外部请求统一通过网关
├── 路由转发：根据路径转发到对应微服务
├── 认证鉴权：JWT Token 验证
├── 限流熔断：基于 Sentinel 的流量控制
├── 日志记录：请求日志、审计日志
└── 跨域处理：CORS 配置

技术栈：
└── Spring Cloud Gateway + Sentinel + Redis
```

**2. oj-user-service (用户服务)**
```java
功能：
├── 用户注册：邮箱/手机号注册
├── 用户登录：JWT Token 生成
├── 用户信息管理：个人资料、头像上传
├── 权限管理：基于角色的访问控制 (RBAC)
└── 社交功能：关注、点赞、评论

数据库表：
├── t_user (用户表)
├── t_role (角色表)
├── t_permission (权限表)
└── t_user_follow (关注关系表)
```

**3. oj-question-service (题目服务)**
```java
功能：
├── 题目管理：增删改查
├── 题目分类：算法、数据库、前端等
├── 难度标记：简单、中等、困难
├── 测试用例管理：输入输出样例
├── 题解管理：官方题解、用户题解
└── 标签系统：动态规划、贪心、二叉树等

数据库表：
├── t_question (题目表)
├── t_test_case (测试用例表)
├── t_question_tag (题目标签表)
├── t_solution (题解表)
└── t_question_category (题目分类表)
```

**4. oj-judge-service (判题服务) ⭐核心**
```java
功能：
├── 代码接收：接收用户提交的代码
├── 任务调度：将判题任务推送到消息队列
├── 代码编译：支持多语言编译
├── 代码执行：Docker 沙箱隔离执行
├── 测试用例校验：对比预期输出
├── 资源监控：CPU、内存、时间监控
├── 结果返回：执行结果、错误信息
└── 判题策略：标准判题、特殊判题 (SPJ)

技术栈：
└── RabbitMQ + Docker + Redis + ThreadPoolExecutor
```

**5. oj-submission-service (提交记录服务)**
```java
功能：
├── 提交历史：保存所有提交记录
├── 代码查看：查看历史提交代码
├── 统计分析：通过率、提交次数
├── 状态查询：实时查询判题状态
└── 数据分页：支持大数据量分页查询

数据库表：
├── t_submission (提交记录表)
└── t_submission_detail (提交详情表)
```

**6. oj-ranking-service (排行榜服务)**
```java
功能：
├── 全局排行榜：基于解题数量
├── 题目排行榜：单题最优解排名
├── 周赛排名：定时更新
└── 实时统计：Redis Sorted Set

技术栈：
└── Redis + Scheduled Tasks
```

### 4.2 微服务通信

#### 4.2.1 同步调用

```java
// 使用 OpenFeign 进行服务间调用
@FeignClient(name = "oj-question-service", fallback = QuestionServiceFallback.class)
public interface QuestionServiceClient {

    @GetMapping("/api/questions/{questionId}")
    QuestionDTO getQuestionById(@PathVariable("questionId") Long questionId);

    @GetMapping("/api/questions/{questionId}/test-cases")
    List<TestCaseDTO> getTestCases(@PathVariable("questionId") Long questionId);
}

// 熔断降级
@Component
public class QuestionServiceFallback implements QuestionServiceClient {
    @Override
    public QuestionDTO getQuestionById(Long questionId) {
        throw new ServiceUnavailableException("题目服务暂时不可用");
    }
}
```

#### 4.2.2 异步通信

```java
// 判题服务发送消息到队列
@Service
public class JudgeTaskProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendJudgeTask(JudgeTaskDTO task) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.JUDGE_EXCHANGE,
            RabbitMQConfig.JUDGE_ROUTING_KEY,
            task
        );
    }
}

// 判题执行器消费队列
@Component
public class JudgeTaskConsumer {

    @RabbitListener(queues = RabbitMQConfig.JUDGE_QUEUE)
    public void handleJudgeTask(JudgeTaskDTO task) {
        // 执行判题逻辑
        judgeExecutor.execute(task);
    }
}
```

---

## 5. 数据库设计

### 5.1 数据库选型

- **主数据库**: MySQL 8.0+ (ACID 保证)
- **缓存**: Redis 7.0+ (热点数据)
- **搜索引擎**: Elasticsearch 8.x (题目全文搜索)
- **对象存储**: MinIO (代码文件、测试用例文件)

### 5.2 核心表结构

#### 5.2.1 用户表 (t_user)

```sql
CREATE TABLE `t_user` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '用户 ID',
  `username` VARCHAR(50) NOT NULL COMMENT '用户名',
  `email` VARCHAR(100) NOT NULL COMMENT '邮箱',
  `password_hash` VARCHAR(255) NOT NULL COMMENT '密码哈希',
  `avatar_url` VARCHAR(500) DEFAULT NULL COMMENT '头像 URL',
  `nickname` VARCHAR(50) DEFAULT NULL COMMENT '昵称',
  `bio` TEXT DEFAULT NULL COMMENT '个人简介',
  `role` VARCHAR(20) NOT NULL DEFAULT 'USER' COMMENT '角色: USER, ADMIN',
  `status` TINYINT NOT NULL DEFAULT 1 COMMENT '状态: 0-禁用, 1-正常',
  `solved_count` INT NOT NULL DEFAULT 0 COMMENT '已解决题目数',
  `submission_count` INT NOT NULL DEFAULT 0 COMMENT '总提交数',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_username` (`username`),
  UNIQUE KEY `uk_email` (`email`),
  KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户表';
```

#### 5.2.2 题目表 (t_question)

```sql
CREATE TABLE `t_question` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '题目 ID',
  `title` VARCHAR(200) NOT NULL COMMENT '题目标题',
  `slug` VARCHAR(200) NOT NULL COMMENT 'URL 友好标识',
  `difficulty` ENUM('EASY', 'MEDIUM', 'HARD') NOT NULL COMMENT '难度',
  `description` TEXT NOT NULL COMMENT '题目描述 (Markdown)',
  `input_format` TEXT DEFAULT NULL COMMENT '输入格式说明',
  `output_format` TEXT DEFAULT NULL COMMENT '输出格式说明',
  `examples` JSON DEFAULT NULL COMMENT '示例 JSON 数组',
  `constraints` TEXT DEFAULT NULL COMMENT '数据范围',
  `time_limit` INT NOT NULL DEFAULT 1000 COMMENT '时间限制 (毫秒)',
  `memory_limit` INT NOT NULL DEFAULT 256 COMMENT '内存限制 (MB)',
  `category_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '分类 ID',
  `tags` JSON DEFAULT NULL COMMENT '标签数组',
  `acceptance_rate` DECIMAL(5, 2) DEFAULT 0.00 COMMENT '通过率',
  `total_submissions` INT NOT NULL DEFAULT 0 COMMENT '总提交数',
  `total_accepted` INT NOT NULL DEFAULT 0 COMMENT '通过数',
  `is_published` TINYINT NOT NULL DEFAULT 0 COMMENT '是否发布: 0-草稿, 1-已发布',
  `created_by` BIGINT UNSIGNED NOT NULL COMMENT '创建者 ID',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_slug` (`slug`),
  KEY `idx_difficulty` (`difficulty`),
  KEY `idx_category` (`category_id`),
  KEY `idx_published` (`is_published`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='题目表';
```

#### 5.2.3 测试用例表 (t_test_case)

```sql
CREATE TABLE `t_test_case` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '测试用例 ID',
  `question_id` BIGINT UNSIGNED NOT NULL COMMENT '题目 ID',
  `input` TEXT NOT NULL COMMENT '输入数据',
  `expected_output` TEXT NOT NULL COMMENT '期望输出',
  `is_sample` TINYINT NOT NULL DEFAULT 0 COMMENT '是否为示例: 0-隐藏, 1-公开',
  `weight` INT NOT NULL DEFAULT 1 COMMENT '权重 (用于评分)',
  `sort_order` INT NOT NULL DEFAULT 0 COMMENT '排序',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  KEY `idx_question` (`question_id`, `sort_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='测试用例表';
```

#### 5.2.4 提交记录表 (t_submission)

```sql
CREATE TABLE `t_submission` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '提交 ID',
  `submission_uuid` VARCHAR(36) NOT NULL COMMENT '提交唯一标识',
  `user_id` BIGINT UNSIGNED NOT NULL COMMENT '用户 ID',
  `question_id` BIGINT UNSIGNED NOT NULL COMMENT '题目 ID',
  `language` VARCHAR(20) NOT NULL COMMENT '编程语言: JAVA, PYTHON, CPP 等',
  `code` TEXT NOT NULL COMMENT '提交代码',
  `status` ENUM(
    'PENDING',      -- 等待判题
    'JUDGING',      -- 判题中
    'ACCEPTED',     -- 通过
    'WRONG_ANSWER', -- 答案错误
    'TIME_LIMIT_EXCEEDED',   -- 超时
    'MEMORY_LIMIT_EXCEEDED', -- 内存超限
    'RUNTIME_ERROR',         -- 运行错误
    'COMPILE_ERROR',         -- 编译错误
    'SYSTEM_ERROR'           -- 系统错误
  ) NOT NULL DEFAULT 'PENDING' COMMENT '判题状态',
  `execution_time` INT DEFAULT NULL COMMENT '执行时间 (毫秒)',
  `memory_usage` INT DEFAULT NULL COMMENT '内存使用 (KB)',
  `error_message` TEXT DEFAULT NULL COMMENT '错误信息',
  `passed_test_cases` INT DEFAULT 0 COMMENT '通过的测试用例数',
  `total_test_cases` INT DEFAULT 0 COMMENT '总测试用例数',
  `score` DECIMAL(5, 2) DEFAULT 0.00 COMMENT '得分',
  `judged_at` DATETIME DEFAULT NULL COMMENT '判题完成时间',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '提交时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_submission_uuid` (`submission_uuid`),
  KEY `idx_user` (`user_id`, `created_at`),
  KEY `idx_question` (`question_id`, `status`),
  KEY `idx_status` (`status`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='提交记录表';
```

#### 5.2.5 判题详情表 (t_submission_detail)

```sql
CREATE TABLE `t_submission_detail` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `submission_id` BIGINT UNSIGNED NOT NULL COMMENT '提交 ID',
  `test_case_id` BIGINT UNSIGNED NOT NULL COMMENT '测试用例 ID',
  `status` ENUM('PASSED', 'FAILED', 'ERROR') NOT NULL COMMENT '状态',
  `execution_time` INT DEFAULT NULL COMMENT '执行时间 (毫秒)',
  `memory_usage` INT DEFAULT NULL COMMENT '内存使用 (KB)',
  `actual_output` TEXT DEFAULT NULL COMMENT '实际输出',
  `error_message` TEXT DEFAULT NULL COMMENT '错误信息',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  KEY `idx_submission` (`submission_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='判题详情表';
```

### 5.3 索引设计策略

```sql
-- 1. 查询用户提交历史
-- 索引：idx_user (user_id, created_at)
SELECT * FROM t_submission
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 20;

-- 2. 查询题目统计
-- 索引：idx_question (question_id, status)
SELECT
  status,
  COUNT(*) as count
FROM t_submission
WHERE question_id = ?
GROUP BY status;

-- 3. 排行榜查询
-- 索引：idx_execution (question_id, status, execution_time)
SELECT
  user_id,
  execution_time,
  memory_usage
FROM t_submission
WHERE question_id = ? AND status = 'ACCEPTED'
ORDER BY execution_time ASC, memory_usage ASC
LIMIT 100;
```

### 5.4 分库分表策略

#### 5.4.1 垂直分库

```
oj-user-db       (用户库)
  └── t_user, t_role, t_permission

oj-question-db   (题目库)
  └── t_question, t_test_case, t_solution

oj-submission-db (提交库) ⭐ 核心，需要分表
  └── t_submission, t_submission_detail
```

#### 5.4.2 水平分表 (提交表)

```java
// 按用户 ID Hash 分表 (16 张表)
public class SubmissionShardingStrategy {

    private static final int TABLE_COUNT = 16;

    public String getTableName(Long userId) {
        int tableIndex = (int) (userId % TABLE_COUNT);
        return String.format("t_submission_%02d", tableIndex);
    }
}

// 表名示例
t_submission_00
t_submission_01
...
t_submission_15
```

**分表依据**：
- 提交记录增长快，单表容易超过千万级
- 按用户 ID 分表，查询用户历史只需访问一张表
- 按题目查询需要扫描多表，但可以用 Redis 缓存统计数据

---

## 6. 核心模块设计

### 6.1 判题服务 (oj-judge-service) 详细设计

#### 6.1.1 模块结构

```
oj-judge-service/
├── src/main/java/com/oj/judge/
│   ├── JudgeApplication.java           # 启动类
│   ├── config/
│   │   ├── RabbitMQConfig.java         # RabbitMQ 配置
│   │   ├── DockerConfig.java           # Docker 客户端配置
│   │   ├── ThreadPoolConfig.java       # 线程池配置
│   │   └── RedisConfig.java            # Redis 配置
│   ├── controller/
│   │   └── JudgeController.java        # 判题接口
│   ├── service/
│   │   ├── JudgeService.java           # 判题服务接口
│   │   ├── impl/
│   │   │   └── JudgeServiceImpl.java   # 判题服务实现
│   │   ├── CodeExecutor.java           # 代码执行器接口
│   │   ├── executor/                   # 各语言执行器
│   │   │   ├── JavaCodeExecutor.java
│   │   │   ├── PythonCodeExecutor.java
│   │   │   ├── CppCodeExecutor.java
│   │   │   └── ...
│   │   └── sandbox/
│   │       ├── DockerSandbox.java      # Docker 沙箱
│   │       └── SandboxResult.java      # 执行结果
│   ├── strategy/
│   │   ├── JudgeStrategy.java          # 判题策略接口
│   │   ├── DefaultJudgeStrategy.java   # 默认判题策略
│   │   └── SpecialJudgeStrategy.java   # 特殊判题策略 (SPJ)
│   ├── consumer/
│   │   └── JudgeTaskConsumer.java      # 消息队列消费者
│   ├── producer/
│   │   └── JudgeResultProducer.java    # 结果发布者
│   ├── monitor/
│   │   ├── ResourceMonitor.java        # 资源监控
│   │   └── PerformanceCollector.java   # 性能收集
│   ├── dto/
│   │   ├── JudgeRequest.java
│   │   ├── JudgeResponse.java
│   │   └── TestCaseResult.java
│   └── exception/
│       ├── CompileException.java
│       ├── RuntimeException.java
│       └── TimeoutException.java
├── src/main/resources/
│   ├── application.yml
│   ├── docker/                         # Docker 镜像 Dockerfile
│   │   ├── java/
│   │   │   ├── Dockerfile.java8
│   │   │   ├── Dockerfile.java11
│   │   │   └── Dockerfile.java17
│   │   ├── python/
│   │   │   ├── Dockerfile.python38
│   │   │   └── Dockerfile.python39
│   │   └── cpp/
│   │       └── Dockerfile.cpp
│   └── templates/                      # 代码模板
│       ├── Main.java.template
│       └── main.py.template
└── pom.xml
```

#### 6.1.2 判题流程图

```
[1] 用户提交代码
      ↓
[2] Controller 接收请求
      ↓
[3] 参数校验
      ↓
[4] 创建提交记录 (状态: PENDING)
      ↓
[5] 发送判题任务到 RabbitMQ
      ↓
[6] 返回提交 UUID 给用户
      ↓
─────────────────────────────────────
      ↓
[7] 消费者从队列获取任务
      ↓
[8] 更新状态为 JUDGING
      ↓
[9] 获取题目信息和测试用例
      ↓
[10] 选择语言执行器
      ↓
[11] Docker 沙箱初始化
      ↓
[12] 代码编译 (如需)
      ├─ 成功 → 继续
      └─ 失败 → 返回 COMPILE_ERROR
      ↓
[13] 循环执行所有测试用例
      ├─ 创建 Docker 容器
      ├─ 挂载输入文件
      ├─ 启动容器并监控
      ├─ 收集输出、时间、内存
      ├─ 对比预期输出
      └─ 记录测试用例结果
      ↓
[14] 判题策略评分
      ↓
[15] 更新提交记录
      ├─ status: ACCEPTED / WRONG_ANSWER / TLE / MLE
      ├─ execution_time: 最大执行时间
      ├─ memory_usage: 最大内存使用
      └─ passed_test_cases / total_test_cases
      ↓
[16] 发送结果到 WebSocket (实时通知)
      ↓
[17] 更新题目统计 (通过率)
      ↓
[18] 清理 Docker 容器
      ↓
[19] 完成
```

#### 6.1.3 核心代码实现

**JudgeController.java**
```java
@RestController
@RequestMapping("/api/judge")
@Slf4j
public class JudgeController {

    @Autowired
    private JudgeService judgeService;

    /**
     * 提交代码进行判题
     */
    @PostMapping("/submit")
    @SentinelResource(value = "judge-submit", blockHandler = "handleBlock")
    public ResponseEntity<JudgeResponse> submit(@RequestBody @Valid JudgeRequest request) {

        log.info("接收判题请求: questionId={}, language={}, userId={}",
            request.getQuestionId(), request.getLanguage(), request.getUserId());

        // 调用判题服务
        String submissionUuid = judgeService.submit(request);

        return ResponseEntity.ok(JudgeResponse.builder()
            .submissionUuid(submissionUuid)
            .status("PENDING")
            .message("代码已提交，正在判题...")
            .build());
    }

    /**
     * 查询判题结果
     */
    @GetMapping("/result/{submissionUuid}")
    public ResponseEntity<SubmissionDTO> getResult(@PathVariable String submissionUuid) {
        SubmissionDTO result = judgeService.getSubmissionResult(submissionUuid);
        return ResponseEntity.ok(result);
    }

    /**
     * 限流降级处理
     */
    public ResponseEntity<JudgeResponse> handleBlock(JudgeRequest request, BlockException ex) {
        return ResponseEntity.status(429)
            .body(JudgeResponse.builder()
                .message("系统繁忙，请稍后再试")
                .build());
    }
}
```

**JudgeServiceImpl.java**
```java
@Service
@Slf4j
public class JudgeServiceImpl implements JudgeService {

    @Autowired
    private SubmissionRepository submissionRepository;

    @Autowired
    private JudgeTaskProducer judgeTaskProducer;

    @Autowired
    private QuestionServiceClient questionServiceClient;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Override
    @Transactional
    public String submit(JudgeRequest request) {

        // 1. 参数校验
        validateRequest(request);

        // 2. 检查代码长度限制
        if (request.getCode().length() > 65535) {
            throw new BusinessException("代码长度超过限制");
        }

        // 3. 检查题目是否存在
        QuestionDTO question = questionServiceClient.getQuestionById(request.getQuestionId());
        if (question == null) {
            throw new BusinessException("题目不存在");
        }

        // 4. 限流检查 (每个用户每分钟最多提交 10 次)
        String rateLimitKey = String.format("judge:rate:%d", request.getUserId());
        Long count = redisTemplate.opsForValue().increment(rateLimitKey);
        if (count == 1) {
            redisTemplate.expire(rateLimitKey, 60, TimeUnit.SECONDS);
        }
        if (count > 10) {
            throw new BusinessException("提交过于频繁，请稍后再试");
        }

        // 5. 创建提交记录
        Submission submission = Submission.builder()
            .submissionUuid(UUID.randomUUID().toString())
            .userId(request.getUserId())
            .questionId(request.getQuestionId())
            .language(request.getLanguage())
            .code(request.getCode())
            .status(SubmissionStatus.PENDING)
            .totalTestCases(question.getTestCaseCount())
            .build();

        submissionRepository.save(submission);

        // 6. 发送判题任务到消息队列
        JudgeTaskDTO task = JudgeTaskDTO.builder()
            .submissionId(submission.getId())
            .submissionUuid(submission.getSubmissionUuid())
            .questionId(request.getQuestionId())
            .userId(request.getUserId())
            .language(request.getLanguage())
            .code(request.getCode())
            .build();

        judgeTaskProducer.sendTask(task);

        log.info("判题任务已发送: submissionUuid={}", submission.getSubmissionUuid());

        return submission.getSubmissionUuid();
    }

    private void validateRequest(JudgeRequest request) {
        if (request.getQuestionId() == null) {
            throw new BusinessException("题目 ID 不能为空");
        }
        if (StringUtils.isBlank(request.getCode())) {
            throw new BusinessException("代码不能为空");
        }
        if (StringUtils.isBlank(request.getLanguage())) {
            throw new BusinessException("编程语言不能为空");
        }
    }
}
```

**JudgeTaskConsumer.java (消息队列消费者)**
```java
@Component
@Slf4j
public class JudgeTaskConsumer {

    @Autowired
    private SubmissionRepository submissionRepository;

    @Autowired
    private QuestionServiceClient questionServiceClient;

    @Autowired
    private CodeExecutorFactory codeExecutorFactory;

    @Autowired
    private JudgeStrategyFactory judgeStrategyFactory;

    @Autowired
    private WebSocketNotifier webSocketNotifier;

    /**
     * 监听判题队列
     */
    @RabbitListener(queues = RabbitMQConfig.JUDGE_QUEUE, concurrency = "5-20")
    public void handleJudgeTask(JudgeTaskDTO task) {

        log.info("开始处理判题任务: submissionUuid={}", task.getSubmissionUuid());

        Submission submission = null;
        try {
            // 1. 更新状态为 JUDGING
            submission = submissionRepository.findBySubmissionUuid(task.getSubmissionUuid());
            submission.setStatus(SubmissionStatus.JUDGING);
            submissionRepository.save(submission);

            // 2. 获取题目和测试用例
            QuestionDTO question = questionServiceClient.getQuestionById(task.getQuestionId());
            List<TestCaseDTO> testCases = questionServiceClient.getTestCases(task.getQuestionId());

            // 3. 选择代码执行器
            CodeExecutor executor = codeExecutorFactory.getExecutor(task.getLanguage());

            // 4. 执行所有测试用例
            List<TestCaseResult> results = new ArrayList<>();
            int passedCount = 0;
            long maxExecutionTime = 0;
            long maxMemoryUsage = 0;

            for (TestCaseDTO testCase : testCases) {

                // 执行单个测试用例
                SandboxResult sandboxResult = executor.execute(
                    task.getCode(),
                    testCase.getInput(),
                    question.getTimeLimit(),
                    question.getMemoryLimit()
                );

                // 判断结果
                boolean passed = sandboxResult.getOutput().trim()
                    .equals(testCase.getExpectedOutput().trim());

                if (passed) {
                    passedCount++;
                }

                // 更新最大值
                maxExecutionTime = Math.max(maxExecutionTime, sandboxResult.getExecutionTime());
                maxMemoryUsage = Math.max(maxMemoryUsage, sandboxResult.getMemoryUsage());

                // 保存测试用例结果
                TestCaseResult result = TestCaseResult.builder()
                    .testCaseId(testCase.getId())
                    .status(passed ? "PASSED" : "FAILED")
                    .executionTime(sandboxResult.getExecutionTime())
                    .memoryUsage(sandboxResult.getMemoryUsage())
                    .actualOutput(sandboxResult.getOutput())
                    .build();

                results.add(result);

                // 如果失败，根据策略决定是否继续
                if (!passed && !question.isShowAllTestCases()) {
                    break; // 遇到第一个失败就停止
                }
            }

            // 5. 应用判题策略
            JudgeStrategy strategy = judgeStrategyFactory.getStrategy(question);
            SubmissionStatus finalStatus = strategy.judge(passedCount, testCases.size(), results);

            // 6. 更新提交记录
            submission.setStatus(finalStatus);
            submission.setPassedTestCases(passedCount);
            submission.setExecutionTime((int) maxExecutionTime);
            submission.setMemoryUsage((int) maxMemoryUsage);
            submission.setJudgedAt(LocalDateTime.now());
            submissionRepository.save(submission);

            // 7. 保存测试用例详情
            saveTestCaseDetails(submission.getId(), results);

            // 8. 发送 WebSocket 通知
            webSocketNotifier.notifySubmissionResult(task.getUserId(), submission);

            // 9. 更新题目统计
            updateQuestionStatistics(task.getQuestionId(), finalStatus);

            log.info("判题完成: submissionUuid={}, status={}, time={}ms, memory={}KB",
                task.getSubmissionUuid(), finalStatus, maxExecutionTime, maxMemoryUsage);

        } catch (CompileException e) {
            // 编译错误
            submission.setStatus(SubmissionStatus.COMPILE_ERROR);
            submission.setErrorMessage(e.getMessage());
            submissionRepository.save(submission);

        } catch (Exception e) {
            // 系统错误
            log.error("判题失败: submissionUuid={}", task.getSubmissionUuid(), e);
            submission.setStatus(SubmissionStatus.SYSTEM_ERROR);
            submission.setErrorMessage("系统错误: " + e.getMessage());
            submissionRepository.save(submission);
        }
    }
}
```

**DockerSandbox.java (Docker 沙箱核心)**
```java
@Component
@Slf4j
public class DockerSandbox {

    @Autowired
    private DockerClient dockerClient;

    /**
     * 在 Docker 容器中执行代码
     */
    public SandboxResult execute(ExecutionRequest request) throws Exception {

        String containerId = null;

        try {
            // 1. 选择镜像
            String image = getImage(request.getLanguage(), request.getVersion());

            // 2. 准备执行脚本
            String script = buildExecutionScript(request);

            // 3. 创建容器配置
            HostConfig hostConfig = HostConfig.newHostConfig()
                .withMemory(request.getMemoryLimit() * 1024 * 1024L)        // MB -> Bytes
                .withMemorySwap(request.getMemoryLimit() * 1024 * 1024L)    // 禁用 swap
                .withCpuQuota(100000L)                                       // 限制 CPU (1 核)
                .withCpuPeriod(100000L)
                .withPidsLimit(100L)                                         // 限制进程数
                .withNetworkMode("none");                                    // 禁用网络

            CreateContainerResponse container = dockerClient.createContainerCmd(image)
                .withHostConfig(hostConfig)
                .withCmd("bash", "-c", script)
                .withStopTimeout(10)
                .withAttachStdout(true)
                .withAttachStderr(true)
                .withUser("sandbox")  // 非 root 用户
                .exec();

            containerId = container.getId();

            // 4. 启动容器
            dockerClient.startContainerCmd(containerId).exec();

            // 5. 等待执行完成 (带超时)
            long startTime = System.currentTimeMillis();
            WaitContainerResultCallback callback = new WaitContainerResultCallback();

            Integer statusCode = dockerClient.waitContainerCmd(containerId)
                .exec(callback)
                .awaitStatusCode(request.getTimeLimit() + 1000, TimeUnit.MILLISECONDS);

            long executionTime = System.currentTimeMillis() - startTime;

            // 6. 检查超时
            if (statusCode == null) {
                // 超时，强制停止容器
                dockerClient.stopContainerCmd(containerId).withTimeout(0).exec();
                throw new TimeoutException("执行超时");
            }

            // 7. 获取输出
            String output = getContainerOutput(containerId);

            // 8. 获取资源使用情况
            Statistics stats = dockerClient.statsCmd(containerId)
                .withNoStream(true)
                .exec(new StatisticsCallback())
                .awaitStats();

            long memoryUsage = stats.getMemoryStats().getUsage() / 1024; // Bytes -> KB

            // 9. 构建结果
            return SandboxResult.builder()
                .output(output)
                .executionTime(executionTime)
                .memoryUsage(memoryUsage)
                .exitCode(statusCode)
                .build();

        } finally {
            // 10. 清理容器
            if (containerId != null) {
                try {
                    dockerClient.removeContainerCmd(containerId)
                        .withForce(true)
                        .exec();
                } catch (Exception e) {
                    log.error("清理容器失败: containerId={}", containerId, e);
                }
            }
        }
    }

    /**
     * 构建执行脚本
     */
    private String buildExecutionScript(ExecutionRequest request) {
        StringBuilder script = new StringBuilder();

        // 1. 创建代码文件
        script.append(String.format("cat > code.%s << 'EOF'\n", getFileSuffix(request.getLanguage())));
        script.append(request.getCode());
        script.append("\nEOF\n");

        // 2. 创建输入文件
        if (StringUtils.isNotBlank(request.getInput())) {
            script.append("cat > input.txt << 'EOF'\n");
            script.append(request.getInput());
            script.append("\nEOF\n");
        }

        // 3. 编译 + 运行命令
        script.append(getExecutionCommand(request.getLanguage(), request.hasInput()));

        return script.toString();
    }

    /**
     * 获取执行命令
     */
    private String getExecutionCommand(String language, boolean hasInput) {
        switch (language.toLowerCase()) {
            case "java":
                return hasInput
                    ? "javac Main.java && timeout 5s java Main < input.txt"
                    : "javac Main.java && timeout 5s java Main";

            case "python":
                return hasInput
                    ? "timeout 5s python3 code.py < input.txt"
                    : "timeout 5s python3 code.py";

            case "cpp":
                return hasInput
                    ? "g++ -O2 -std=c++17 code.cpp -o code && timeout 5s ./code < input.txt"
                    : "g++ -O2 -std=c++17 code.cpp -o code && timeout 5s ./code";

            default:
                throw new UnsupportedLanguageException("不支持的语言: " + language);
        }
    }

    /**
     * 获取容器输出
     */
    private String getContainerOutput(String containerId) throws Exception {

        LogContainerCallback callback = new LogContainerCallback();

        dockerClient.logContainerCmd(containerId)
            .withStdOut(true)
            .withStdErr(true)
            .exec(callback)
            .awaitCompletion();

        String output = callback.toString();

        // 限制输出大小
        if (output.length() > 10000) {
            output = output.substring(0, 10000) + "\n...(输出过长，已截断)";
        }

        return output;
    }

    private static class StatisticsCallback extends ResultCallbackTemplate<StatisticsCallback, Statistics> {
        private Statistics stats;

        @Override
        public void onNext(Statistics stats) {
            this.stats = stats;
        }

        public Statistics awaitStats() throws InterruptedException {
            awaitCompletion();
            return stats;
        }
    }
}
```

---

## 7. 代码执行引擎

### 7.1 支持的语言

| 语言 | 编译 | 运行时 | Docker 镜像 | 文件后缀 |
|-----|------|--------|------------|---------|
| Java | ✅ | JVM | openjdk:17-slim | .java |
| Python | ❌ | CPython | python:3.9-slim | .py |
| C++ | ✅ | Native | gcc:13 | .cpp |
| C | ✅ | Native | gcc:13 | .c |
| JavaScript | ❌ | Node.js | node:20-slim | .js |
| Go | ✅ | Native | golang:1.21 | .go |
| Rust | ✅ | Native | rust:1.75 | .rs |

### 7.2 语言执行器工厂

```java
@Component
public class CodeExecutorFactory {

    @Autowired
    private Map<String, CodeExecutor> executors;

    public CodeExecutor getExecutor(String language) {
        CodeExecutor executor = executors.get(language.toLowerCase() + "Executor");

        if (executor == null) {
            throw new UnsupportedLanguageException("不支持的语言: " + language);
        }

        return executor;
    }
}

// Spring 自动注入所有 CodeExecutor Bean
@Component("javaExecutor")
public class JavaCodeExecutor implements CodeExecutor { ... }

@Component("pythonExecutor")
public class PythonCodeExecutor implements CodeExecutor { ... }

@Component("cppExecutor")
public class CppCodeExecutor implements CodeExecutor { ... }
```

### 7.3 Java 执行器示例

```java
@Component("javaExecutor")
@Slf4j
public class JavaCodeExecutor implements CodeExecutor {

    @Autowired
    private DockerSandbox dockerSandbox;

    @Override
    public SandboxResult execute(String code, String input, int timeLimit, int memoryLimit) {

        // 1. 包装代码 (确保类名为 Main)
        String wrappedCode = wrapCode(code);

        // 2. 构建执行请求
        ExecutionRequest request = ExecutionRequest.builder()
            .language("java")
            .version("17")
            .code(wrappedCode)
            .input(input)
            .timeLimit(timeLimit)
            .memoryLimit(memoryLimit)
            .build();

        // 3. 在沙箱中执行
        return dockerSandbox.execute(request);
    }

    /**
     * 包装用户代码为标准格式
     */
    private String wrapCode(String userCode) {

        // 如果用户已经写了 public class Main，直接使用
        if (userCode.contains("public class Main")) {
            return userCode;
        }

        // 否则，包装用户代码
        StringBuilder wrapped = new StringBuilder();
        wrapped.append("import java.util.*;\n");
        wrapped.append("import java.io.*;\n\n");
        wrapped.append("public class Main {\n");
        wrapped.append("    public static void main(String[] args) throws Exception {\n");
        wrapped.append("        Solution solution = new Solution();\n");
        wrapped.append("        // 用户需要在 Solution 类中实现算法\n");
        wrapped.append("    }\n");
        wrapped.append("}\n\n");
        wrapped.append("class Solution {\n");
        wrapped.append(userCode);
        wrapped.append("\n}\n");

        return wrapped.toString();
    }

    @Override
    public String getLanguage() {
        return "java";
    }
}
```

### 7.4 Docker 镜像 Dockerfile

**Java 镜像 (Dockerfile.java17)**
```dockerfile
FROM openjdk:17-slim

# 安装必要工具
RUN apt-get update && apt-get install -y \
    timeout \
    && rm -rf /var/lib/apt/lists/*

# 创建非 root 用户
RUN useradd -m -u 1000 -s /bin/bash sandbox

# 设置工作目录
WORKDIR /sandbox

# 切换用户
USER sandbox

CMD ["/bin/bash"]
```

**Python 镜像 (Dockerfile.python39)**
```dockerfile
FROM python:3.9-slim

# 安装常用库
RUN pip install --no-cache-dir \
    numpy \
    scipy \
    pandas

# 创建非 root 用户
RUN useradd -m -u 1000 -s /bin/bash sandbox

WORKDIR /sandbox
USER sandbox

CMD ["/bin/bash"]
```

**C++ 镜像 (Dockerfile.cpp)**
```dockerfile
FROM gcc:13

# 创建非 root 用户
RUN useradd -m -u 1000 -s /bin/bash sandbox

WORKDIR /sandbox
USER sandbox

CMD ["/bin/bash"]
```

### 7.5 资源监控

```java
@Component
public class ResourceMonitor {

    /**
     * 监控容器资源使用
     */
    public ResourceMetrics monitor(String containerId, DockerClient dockerClient) {

        Statistics stats = dockerClient.statsCmd(containerId)
            .withNoStream(true)
            .exec(new StatisticsCallback())
            .awaitStats();

        // 内存使用
        MemoryStatsConfig memStats = stats.getMemoryStats();
        long memoryUsage = memStats.getUsage() / 1024; // KB
        long memoryLimit = memStats.getLimit() / 1024; // KB

        // CPU 使用
        CpuStatsConfig cpuStats = stats.getCpuStats();
        CpuStatsConfig preCpuStats = stats.getPreCpuStats();

        long cpuDelta = cpuStats.getCpuUsage().getTotalUsage() -
                        preCpuStats.getCpuUsage().getTotalUsage();
        long systemDelta = cpuStats.getSystemCpuUsage() -
                           preCpuStats.getSystemCpuUsage();

        double cpuPercent = 0.0;
        if (systemDelta > 0) {
            cpuPercent = (double) cpuDelta / systemDelta * 100.0;
        }

        return ResourceMetrics.builder()
            .memoryUsage(memoryUsage)
            .memoryLimit(memoryLimit)
            .cpuPercent(cpuPercent)
            .build();
    }
}
```

---

## 8. 高并发解决方案

### 8.1 并发场景分析

| 场景 | QPS | 特点 | 挑战 |
|-----|-----|------|------|
| **代码提交** | 1000+ | 突发性高 | 瞬时大量任务 |
| **判题执行** | 500+ | CPU 密集 | Docker 容器创建慢 |
| **结果查询** | 5000+ | 读多写少 | 数据库压力 |
| **排行榜** | 3000+ | 实时性要求高 | 计算密集 |

### 8.2 高并发架构

```
┌─────────────────────────────────────────────────────────────┐
│                     1. 接入层优化                            │
├─────────────────────────────────────────────────────────────┤
│ • Nginx 负载均衡 (多台 Gateway)                              │
│ • 限流: Sentinel (QPS 限制、熔断降级)                        │
│ • CDN: 静态资源加速                                          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     2. 应用层优化                            │
├─────────────────────────────────────────────────────────────┤
│ • 无状态服务: 水平扩展                                       │
│ • 连接池: HikariCP (数据库)、Lettuce (Redis)                │
│ • 线程池: 业务隔离、异步处理                                 │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     3. 消息队列削峰                          │
├─────────────────────────────────────────────────────────────┤
│ • RabbitMQ: 异步判题队列                                     │
│ • 死信队列: 失败重试                                         │
│ • 优先级队列: VIP 用户优先                                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     4. 数据层优化                            │
├─────────────────────────────────────────────────────────────┤
│ • Redis 缓存: 热点题目、用户信息                             │
│ • 读写分离: 主从复制                                         │
│ • 分库分表: 提交记录分表                                     │
│ • 索引优化: 覆盖索引、联合索引                               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     5. 容器池化                              │
├─────────────────────────────────────────────────────────────┤
│ • Docker 容器预创建: 避免每次创建开销                       │
│ • 容器复用: 执行完清理后复用                                 │
│ • 容器生命周期管理: 定期回收                                 │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 限流策略

#### 8.3.1 网关层限流 (Sentinel)

```java
@Configuration
public class SentinelConfig {

    @PostConstruct
    public void initFlowRules() {
        List<FlowRule> rules = new ArrayList<>();

        // 1. 全局限流: 1000 QPS
        FlowRule globalRule = new FlowRule();
        globalRule.setResource("judge-submit");
        globalRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        globalRule.setCount(1000);
        rules.add(globalRule);

        // 2. 用户维度限流: 每用户 10 QPS
        FlowRule userRule = new FlowRule();
        userRule.setResource("judge-submit");
        userRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        userRule.setCount(10);
        userRule.setLimitApp("user");
        rules.add(userRule);

        FlowRuleManager.loadRules(rules);
    }
}
```

#### 8.3.2 应用层限流 (Redis + Lua)

```java
@Component
public class RateLimiter {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    /**
     * 滑动窗口限流
     * @param key 限流键
     * @param limit 限制次数
     * @param window 时间窗口 (秒)
     */
    public boolean tryAcquire(String key, int limit, int window) {

        String luaScript =
            "local key = KEYS[1] " +
            "local limit = tonumber(ARGV[1]) " +
            "local window = tonumber(ARGV[2]) " +
            "local current = tonumber(redis.call('get', key) or '0') " +
            "if current >= limit then " +
            "    return 0 " +
            "else " +
            "    redis.call('incr', key) " +
            "    if current == 0 then " +
            "        redis.call('expire', key, window) " +
            "    end " +
            "    return 1 " +
            "end";

        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Collections.singletonList(key),
            limit,
            window
        );

        return result != null && result == 1;
    }
}

// 使用示例
String rateLimitKey = String.format("rate:user:%d", userId);
if (!rateLimiter.tryAcquire(rateLimitKey, 10, 60)) {
    throw new TooManyRequestsException("提交过于频繁");
}
```

### 8.4 消息队列配置

#### 8.4.1 RabbitMQ 配置

```java
@Configuration
public class RabbitMQConfig {

    public static final String JUDGE_EXCHANGE = "oj.judge.exchange";
    public static final String JUDGE_QUEUE = "oj.judge.queue";
    public static final String JUDGE_ROUTING_KEY = "oj.judge.task";

    public static final String JUDGE_DLX_EXCHANGE = "oj.judge.dlx.exchange";
    public static final String JUDGE_DLX_QUEUE = "oj.judge.dlx.queue";

    /**
     * 判题交换机
     */
    @Bean
    public DirectExchange judgeExchange() {
        return new DirectExchange(JUDGE_EXCHANGE, true, false);
    }

    /**
     * 判题队列 (带死信)
     */
    @Bean
    public Queue judgeQueue() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-dead-letter-exchange", JUDGE_DLX_EXCHANGE);
        args.put("x-dead-letter-routing-key", "dlx");
        args.put("x-message-ttl", 300000); // 5 分钟 TTL

        return new Queue(JUDGE_QUEUE, true, false, false, args);
    }

    /**
     * 绑定
     */
    @Bean
    public Binding judgeBinding() {
        return BindingBuilder
            .bind(judgeQueue())
            .to(judgeExchange())
            .with(JUDGE_ROUTING_KEY);
    }

    /**
     * 死信交换机
     */
    @Bean
    public DirectExchange dlxExchange() {
        return new DirectExchange(JUDGE_DLX_EXCHANGE, true, false);
    }

    /**
     * 死信队列
     */
    @Bean
    public Queue dlxQueue() {
        return new Queue(JUDGE_DLX_QUEUE, true);
    }

    /**
     * 死信绑定
     */
    @Bean
    public Binding dlxBinding() {
        return BindingBuilder
            .bind(dlxQueue())
            .to(dlxExchange())
            .with("dlx");
    }

    /**
     * 消息转换器 (JSON)
     */
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

#### 8.4.2 动态消费者配置

```java
/**
 * 动态调整消费者数量
 * 根据队列积压情况自动扩缩容
 */
@Component
@Slf4j
public class DynamicConsumerManager {

    @Autowired
    private RabbitAdmin rabbitAdmin;

    @Scheduled(fixedDelay = 10000) // 每 10 秒检查一次
    public void adjustConsumers() {

        Properties queueProperties = rabbitAdmin.getQueueProperties(RabbitMQConfig.JUDGE_QUEUE);

        if (queueProperties != null) {
            Integer messageCount = (Integer) queueProperties.get("QUEUE_MESSAGE_COUNT");

            log.info("当前队列积压: {} 条消息", messageCount);

            // 根据积压情况调整消费者
            // 这里可以动态调整 @RabbitListener 的 concurrency
            // 实际生产中可以通过 Kubernetes HPA 自动扩容 Pod
        }
    }
}
```

### 8.5 缓存策略

#### 8.5.1 多级缓存

```
┌─────────────────────────────────────┐
│      L1: 本地缓存 (Caffeine)         │
│      • 题目详情 (TTL: 5分钟)         │
│      • 用户信息 (TTL: 1分钟)         │
└─────────────────────────────────────┘
                  ↓ Miss
┌─────────────────────────────────────┐
│      L2: Redis 缓存                  │
│      • 题目详情 (TTL: 1小时)         │
│      • 测试用例 (TTL: 1小时)         │
│      • 排行榜 (实时更新)             │
└─────────────────────────────────────┘
                  ↓ Miss
┌─────────────────────────────────────┐
│      L3: MySQL 数据库                │
└─────────────────────────────────────┘
```

#### 8.5.2 缓存代码实现

```java
@Service
public class QuestionCacheService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private QuestionRepository questionRepository;

    // 本地缓存 (Caffeine)
    private Cache<Long, QuestionDTO> localCache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build();

    /**
     * 获取题目 (多级缓存)
     */
    public QuestionDTO getQuestion(Long questionId) {

        // 1. 查询本地缓存
        QuestionDTO question = localCache.getIfPresent(questionId);
        if (question != null) {
            return question;
        }

        // 2. 查询 Redis
        String redisKey = String.format("question:%d", questionId);
        question = (QuestionDTO) redisTemplate.opsForValue().get(redisKey);
        if (question != null) {
            localCache.put(questionId, question);
            return question;
        }

        // 3. 查询数据库
        Question entity = questionRepository.findById(questionId)
            .orElseThrow(() -> new NotFoundException("题目不存在"));

        question = convertToDTO(entity);

        // 4. 写入缓存
        redisTemplate.opsForValue().set(redisKey, question, 1, TimeUnit.HOURS);
        localCache.put(questionId, question);

        return question;
    }

    /**
     * 缓存预热
     */
    @PostConstruct
    public void warmUp() {
        // 预加载热门题目
        List<Question> hotQuestions = questionRepository.findTop100ByOrderByTotalSubmissionsDesc();

        for (Question question : hotQuestions) {
            String redisKey = String.format("question:%d", question.getId());
            redisTemplate.opsForValue().set(
                redisKey,
                convertToDTO(question),
                1,
                TimeUnit.HOURS
            );
        }

        log.info("缓存预热完成: {} 个热门题目", hotQuestions.size());
    }
}
```

### 8.6 Docker 容器池

#### 8.6.1 容器池设计

```java
/**
 * Docker 容器对象池
 * 复用容器，避免频繁创建销毁
 */
@Component
@Slf4j
public class DockerContainerPool {

    @Autowired
    private DockerClient dockerClient;

    // 容器池: key=镜像名, value=可用容器队列
    private Map<String, BlockingQueue<String>> containerPools = new ConcurrentHashMap<>();

    // 池配置
    private static final int MIN_POOL_SIZE = 5;
    private static final int MAX_POOL_SIZE = 50;

    /**
     * 初始化容器池
     */
    @PostConstruct
    public void init() {

        String[] images = {"java:17", "python:3.9", "cpp:gcc13"};

        for (String image : images) {
            containerPools.put(image, new LinkedBlockingQueue<>(MAX_POOL_SIZE));

            // 预创建容器
            for (int i = 0; i < MIN_POOL_SIZE; i++) {
                String containerId = createContainer(image);
                containerPools.get(image).offer(containerId);
            }

            log.info("容器池初始化完成: image={}, size={}", image, MIN_POOL_SIZE);
        }
    }

    /**
     * 获取容器
     */
    public String acquireContainer(String image) throws InterruptedException {

        BlockingQueue<String> pool = containerPools.get(image);

        if (pool == null) {
            throw new IllegalArgumentException("不支持的镜像: " + image);
        }

        // 尝试从池中获取
        String containerId = pool.poll();

        if (containerId == null) {
            // 池中无可用容器，动态创建
            log.warn("容器池耗尽，动态创建新容器: image={}", image);
            containerId = createContainer(image);
        }

        return containerId;
    }

    /**
     * 归还容器
     */
    public void releaseContainer(String image, String containerId) {

        try {
            // 清理容器 (删除文件、重置状态)
            cleanContainer(containerId);

            // 归还到池中
            BlockingQueue<String> pool = containerPools.get(image);

            if (!pool.offer(containerId)) {
                // 池已满，销毁容器
                dockerClient.removeContainerCmd(containerId).withForce(true).exec();
                log.info("容器池已满，销毁容器: containerId={}", containerId);
            }

        } catch (Exception e) {
            log.error("归还容器失败: containerId={}", containerId, e);
            // 发生异常，直接销毁
            try {
                dockerClient.removeContainerCmd(containerId).withForce(true).exec();
            } catch (Exception ex) {
                log.error("销毁容器失败: containerId={}", containerId, ex);
            }
        }
    }

    /**
     * 创建容器
     */
    private String createContainer(String image) {

        HostConfig hostConfig = HostConfig.newHostConfig()
            .withMemory(256 * 1024 * 1024L)
            .withCpuQuota(100000L)
            .withNetworkMode("none");

        CreateContainerResponse container = dockerClient.createContainerCmd(image)
            .withHostConfig(hostConfig)
            .withUser("sandbox")
            .exec();

        return container.getId();
    }

    /**
     * 清理容器 (删除临时文件)
     */
    private void cleanContainer(String containerId) {
        dockerClient.execCreateCmd(containerId)
            .withCmd("rm", "-rf", "/sandbox/*")
            .exec();
    }

    /**
     * 定期清理过期容器
     */
    @Scheduled(fixedDelay = 600000) // 每 10 分钟
    public void cleanExpiredContainers() {

        for (Map.Entry<String, BlockingQueue<String>> entry : containerPools.entrySet()) {

            String image = entry.getKey();
            BlockingQueue<String> pool = entry.getValue();

            // 如果池中容器超过最小值，移除多余的
            while (pool.size() > MIN_POOL_SIZE) {
                String containerId = pool.poll();
                if (containerId != null) {
                    try {
                        dockerClient.removeContainerCmd(containerId).withForce(true).exec();
                        log.info("清理过期容器: image={}, containerId={}", image, containerId);
                    } catch (Exception e) {
                        log.error("清理容器失败: containerId={}", containerId, e);
                    }
                }
            }
        }
    }
}
```

### 8.7 线程池配置

```java
@Configuration
public class ThreadPoolConfig {

    /**
     * 判题执行线程池
     */
    @Bean(name = "judgeExecutor")
    public ThreadPoolExecutor judgeExecutor() {

        int corePoolSize = Runtime.getRuntime().availableProcessors() * 2;
        int maxPoolSize = corePoolSize * 4;

        return new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            60L,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            new ThreadFactoryBuilder().setNameFormat("judge-pool-%d").build(),
            new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略：调用者运行
        );
    }

    /**
     * 异步任务线程池
     */
    @Bean(name = "asyncExecutor")
    public Executor asyncExecutor() {

        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();

        return executor;
    }
}
```

---

## 9. 性能监控与资源管理

### 9.1 监控架构

```
┌─────────────────────────────────────────────────────────────┐
│                   应用层 (Micrometer)                        │
│  • 自定义指标收集                                            │
│  • JVM 监控 (GC、内存、线程)                                 │
│  • 接口响应时间                                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   时序数据库 (Prometheus)                     │
│  • 指标存储                                                  │
│  • 告警规则                                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   可视化 (Grafana)                           │
│  • 实时监控大盘                                              │
│  • 告警通知                                                  │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 自定义指标

```java
@Component
public class JudgeMetrics {

    private final Counter submissionCounter;
    private final Timer judgeDurationTimer;
    private final Gauge activeContainersGauge;

    public JudgeMetrics(MeterRegistry registry) {

        // 提交计数器
        this.submissionCounter = Counter.builder("judge.submission.total")
            .description("总提交数")
            .tag("type", "all")
            .register(registry);

        // 判题耗时
        this.judgeDurationTimer = Timer.builder("judge.duration")
            .description("判题耗时")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        // 活跃容器数
        this.activeContainersGauge = Gauge.builder("judge.containers.active",
            dockerContainerPool,
            pool -> pool.getActiveCount())
            .description("活跃容器数")
            .register(registry);
    }

    public void recordSubmission() {
        submissionCounter.increment();
    }

    public void recordJudgeDuration(long duration) {
        judgeDurationTimer.record(duration, TimeUnit.MILLISECONDS);
    }
}
```

### 9.3 资源限制

```java
/**
 * 全局资源管理器
 * 防止资源耗尽
 */
@Component
public class ResourceManager {

    @Autowired
    private DockerClient dockerClient;

    // 最大并发判题数
    private Semaphore judgeSemaphore = new Semaphore(100);

    /**
     * 获取判题许可
     */
    public boolean acquireJudgePermit(long timeout, TimeUnit unit) {
        try {
            return judgeSemaphore.tryAcquire(timeout, unit);
        } catch (InterruptedException e) {
            return false;
        }
    }

    /**
     * 释放判题许可
     */
    public void releaseJudgePermit() {
        judgeSemaphore.release();
    }

    /**
     * 监控系统资源
     */
    @Scheduled(fixedDelay = 30000) // 每 30 秒
    public void monitorSystemResources() {

        // 1. 检查 Docker 容器数量
        List<Container> containers = dockerClient.listContainersCmd()
            .withShowAll(true)
            .exec();

        if (containers.size() > 200) {
            log.warn("容器数量过多: {}", containers.size());
            // 触发告警
        }

        // 2. 检查 JVM 内存
        Runtime runtime = Runtime.getRuntime();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        long maxMemory = runtime.maxMemory();
        double memoryUsage = (double) usedMemory / maxMemory * 100;

        if (memoryUsage > 80) {
            log.warn("JVM 内存使用率过高: {}%", memoryUsage);
            // 触发 GC 或告警
        }

        // 3. 检查线程数
        ThreadGroup rootGroup = Thread.currentThread().getThreadGroup();
        int threadCount = rootGroup.activeCount();

        if (threadCount > 500) {
            log.warn("线程数过多: {}", threadCount);
        }
    }
}
```

---

## 10. 安全机制

### 10.1 安全威胁模型

| 威胁 | 风险等级 | 防护措施 |
|-----|---------|---------|
| **恶意代码执行** | 🔴 高 | Docker 沙箱、禁用网络、资源限制 |
| **无限循环/递归** | 🔴 高 | 时间限制、内存限制、进程数限制 |
| **文件系统攻击** | 🟡 中 | 非 root 用户、只读文件系统 |
| **网络攻击** | 🟡 中 | 禁用网络、防火墙 |
| **SQL 注入** | 🔴 高 | 参数化查询、ORM 框架 |
| **XSS 攻击** | 🟡 中 | 输出转义、CSP 策略 |
| **CSRF 攻击** | 🟡 中 | CSRF Token、SameSite Cookie |
| **DDoS 攻击** | 🔴 高 | 限流、熔断、WAF |

### 10.2 Docker 安全配置

```java
/**
 * 安全的 Docker 容器配置
 */
public HostConfig createSecureHostConfig() {

    return HostConfig.newHostConfig()
        // 1. 资源限制
        .withMemory(256 * 1024 * 1024L)              // 内存限制 256MB
        .withMemorySwap(256 * 1024 * 1024L)          // 禁用 swap
        .withCpuQuota(100000L)                        // CPU 限制 1 核
        .withCpuPeriod(100000L)
        .withPidsLimit(100L)                          // 进程数限制

        // 2. 网络隔离
        .withNetworkMode("none")                      // 禁用网络

        // 3. 文件系统限制
        .withReadonlyRootfs(false)                    // 允许写入 /tmp
        .withTmpFs(Map.of("/tmp", "rw,noexec,nosuid,size=100m"))

        // 4. 安全选项
        .withSecurityOpts(List.of(
            "no-new-privileges",                      // 禁止提权
            "seccomp=default"                         // 系统调用过滤
        ))

        // 5. 其他限制
        .withUlimits(List.of(
            new Ulimit("nofile", 100L, 100L),        // 文件描述符限制
            new Ulimit("nproc", 50L, 50L)            // 进程数限制
        ));
}
```

### 10.3 代码检查

```java
@Component
public class CodeSecurityChecker {

    // 危险关键字黑名单
    private static final Set<String> DANGEROUS_KEYWORDS = Set.of(
        "Runtime.getRuntime()",
        "ProcessBuilder",
        "System.exit",
        "File(",
        "FileWriter",
        "FileOutputStream",
        "Socket",
        "ServerSocket",
        "URLConnection",
        "exec(",
        "Runtime.exec"
    );

    /**
     * 检查代码是否包含危险操作
     */
    public void checkCode(String code, String language) {

        if ("java".equalsIgnoreCase(language)) {
            checkJavaCode(code);
        }
        // 其他语言...
    }

    private void checkJavaCode(String code) {

        for (String keyword : DANGEROUS_KEYWORDS) {
            if (code.contains(keyword)) {
                throw new SecurityException("代码包含危险操作: " + keyword);
            }
        }

        // 额外检查: 反射
        if (code.contains("Class.forName") || code.contains("getDeclaredMethod")) {
            throw new SecurityException("禁止使用反射");
        }
    }
}
```

### 10.4 认证授权

```java
/**
 * JWT Token 生成与验证
 */
@Component
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private long expiration;

    /**
     * 生成 Token
     */
    public String generateToken(UserDetails userDetails) {

        Map<String, Object> claims = new HashMap<>();
        claims.put("userId", ((CustomUserDetails) userDetails).getUserId());
        claims.put("role", userDetails.getAuthorities());

        return Jwts.builder()
            .setClaims(claims)
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration * 1000))
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
    }

    /**
     * 验证 Token
     */
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(secret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

---

## 11. API 设计

### 11.1 RESTful API 规范

**基础路径**: `https://api.oj.com/v1`

### 11.2 核心接口

#### 11.2.1 用户相关

```http
POST   /api/v1/auth/register      # 注册
POST   /api/v1/auth/login         # 登录
POST   /api/v1/auth/logout        # 登出
GET    /api/v1/users/me           # 获取当前用户信息
PUT    /api/v1/users/me           # 更新用户信息
GET    /api/v1/users/{userId}     # 获取用户详情
```

**登录接口示例**:
```json
POST /api/v1/auth/login

Request:
{
  "username": "user123",
  "password": "password123"
}

Response (200 OK):
{
  "code": 0,
  "message": "登录成功",
  "data": {
    "token": "eyJhbGciOiJIUzUxMiJ9...",
    "user": {
      "id": 10001,
      "username": "user123",
      "email": "user@example.com",
      "role": "USER"
    }
  }
}
```

#### 11.2.2 题目相关

```http
GET    /api/v1/questions                    # 题目列表 (分页)
GET    /api/v1/questions/{questionId}       # 题目详情
POST   /api/v1/questions                    # 创建题目 (管理员)
PUT    /api/v1/questions/{questionId}       # 更新题目 (管理员)
DELETE /api/v1/questions/{questionId}       # 删除题目 (管理员)
GET    /api/v1/questions/{questionId}/test-cases  # 获取测试用例
```

**题目列表接口示例**:
```json
GET /api/v1/questions?page=1&size=20&difficulty=MEDIUM&tag=动态规划

Response (200 OK):
{
  "code": 0,
  "message": "成功",
  "data": {
    "total": 156,
    "page": 1,
    "size": 20,
    "items": [
      {
        "id": 1,
        "title": "两数之和",
        "slug": "two-sum",
        "difficulty": "EASY",
        "acceptanceRate": 48.5,
        "totalSubmissions": 12345,
        "tags": ["数组", "哈希表"],
        "isSolved": true
      }
    ]
  }
}
```

#### 11.2.3 判题相关

```http
POST   /api/v1/judge/submit                 # 提交代码
GET    /api/v1/judge/result/{submissionUuid} # 查询判题结果
GET    /api/v1/submissions                   # 我的提交历史
GET    /api/v1/submissions/{submissionId}    # 提交详情
```

**提交代码接口示例**:
```json
POST /api/v1/judge/submit

Request:
{
  "questionId": 1,
  "language": "JAVA",
  "code": "class Solution {\n    public int[] twoSum(int[] nums, int target) {\n        ...\n    }\n}"
}

Response (200 OK):
{
  "code": 0,
  "message": "代码已提交",
  "data": {
    "submissionUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "status": "PENDING"
  }
}
```

**查询结果接口示例**:
```json
GET /api/v1/judge/result/a1b2c3d4-e5f6-7890-abcd-ef1234567890

Response (200 OK):
{
  "code": 0,
  "message": "成功",
  "data": {
    "submissionUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "status": "ACCEPTED",
    "executionTime": 245,
    "memoryUsage": 12800,
    "passedTestCases": 58,
    "totalTestCases": 58,
    "language": "JAVA",
    "submittedAt": "2024-01-15T10:30:00Z",
    "judgedAt": "2024-01-15T10:30:05Z"
  }
}
```

#### 11.2.4 排行榜

```http
GET    /api/v1/ranking/global               # 全局排行榜
GET    /api/v1/ranking/question/{questionId} # 单题排行榜
```

### 11.3 WebSocket 实时通知

```
ws://api.oj.com/v1/ws/judge

// 连接后订阅判题结果
{
  "action": "subscribe",
  "submissionUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}

// 服务端推送进度
{
  "type": "progress",
  "submissionUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "JUDGING",
  "progress": 50,
  "message": "正在执行第 29/58 个测试用例"
}

// 服务端推送最终结果
{
  "type": "result",
  "submissionUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "ACCEPTED",
  "executionTime": 245,
  "memoryUsage": 12800
}
```

---

## 12. 部署方案

### 12.1 部署架构

```
┌──────────────────────────────────────────────────────────────┐
│                         负载均衡层                            │
│  ┌────────────┐         ┌────────────┐                       │
│  │  Nginx 1   │         │  Nginx 2   │                       │
│  │  (主节点)  │◄───────►│  (备节点)  │  (Keepalived)         │
│  └────────────┘         └────────────┘                       │
└──────────────────────────────────────────────────────────────┘
              │                       │
              └───────────┬───────────┘
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                      Kubernetes 集群                          │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Namespace: oj-prod                                     │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  Deployment: oj-gateway (副本: 3)                      │  │
│  │  Deployment: oj-user-service (副本: 2)                 │  │
│  │  Deployment: oj-question-service (副本: 2)             │  │
│  │  Deployment: oj-judge-service (副本: 5)                │  │
│  │  Deployment: oj-submission-service (副本: 2)           │  │
│  │  Deployment: oj-ranking-service (副本: 2)              │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  Service: ClusterIP (内部通信)                          │  │
│  │  Ingress: HTTPS (外部访问)                              │  │
│  │  ConfigMap: 配置管理                                    │  │
│  │  Secret: 敏感信息                                       │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
              │                       │
              ▼                       ▼
┌──────────────────────┐  ┌──────────────────────┐
│  有状态服务 (外部)   │  │  Docker 判题集群     │
├──────────────────────┤  ├──────────────────────┤
│ • MySQL (主从)       │  │ • Docker Host 1      │
│ • Redis (哨兵)       │  │ • Docker Host 2      │
│ • RabbitMQ (集群)    │  │ • Docker Host 3      │
│ • Nacos (集群)       │  │   (DinD 或独立节点)  │
└──────────────────────┘  └──────────────────────┘
```

### 12.2 Docker Compose (单机开发环境)

```yaml
# docker-compose.yml
version: '3.8'

services:
  # MySQL
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: oj_db
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - oj-network

  # Redis
  redis:
    image: redis:7.0
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - oj-network

  # RabbitMQ
  rabbitmq:
    image: rabbitmq:3.12-management
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
    ports:
      - "5672:5672"
      - "15672:15672"  # 管理界面
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    networks:
      - oj-network

  # Nacos
  nacos:
    image: nacos/nacos-server:v2.3.0
    environment:
      MODE: standalone
      SPRING_DATASOURCE_PLATFORM: mysql
      MYSQL_SERVICE_HOST: mysql
      MYSQL_SERVICE_DB_NAME: nacos
      MYSQL_SERVICE_USER: root
      MYSQL_SERVICE_PASSWORD: root123
    ports:
      - "8848:8848"
    depends_on:
      - mysql
    networks:
      - oj-network

  # Gateway
  oj-gateway:
    build: ./oj-gateway
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
      NACOS_SERVER_ADDR: nacos:8848
    depends_on:
      - nacos
    networks:
      - oj-network

  # Judge Service
  oj-judge-service:
    build: ./oj-judge-service
    environment:
      SPRING_PROFILES_ACTIVE: prod
      NACOS_SERVER_ADDR: nacos:8848
      DOCKER_HOST: tcp://docker-dind:2376
      DOCKER_TLS_VERIFY: 0
    depends_on:
      - nacos
      - rabbitmq
      - redis
    networks:
      - oj-network

  # Docker-in-Docker (判题沙箱)
  docker-dind:
    image: docker:24-dind
    privileged: true
    environment:
      DOCKER_TLS_CERTDIR: ""
    volumes:
      - docker-data:/var/lib/docker
    networks:
      - oj-network

volumes:
  mysql-data:
  redis-data:
  rabbitmq-data:
  docker-data:

networks:
  oj-network:
    driver: bridge
```

### 12.3 Kubernetes 部署 (生产环境)

**oj-judge-service Deployment**
```yaml
# k8s/oj-judge-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oj-judge-service
  namespace: oj-prod
spec:
  replicas: 5
  selector:
    matchLabels:
      app: oj-judge-service
  template:
    metadata:
      labels:
        app: oj-judge-service
    spec:
      containers:
      - name: oj-judge-service
        image: registry.oj.com/oj-judge-service:1.0.0
        ports:
        - containerPort: 8083
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: NACOS_SERVER_ADDR
          value: "nacos-service:8848"
        - name: RABBITMQ_HOST
          value: "rabbitmq-service"
        - name: REDIS_HOST
          value: "redis-service"
        resources:
          requests:
            cpu: "1000m"
            memory: "2Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8083
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8083
          initialDelaySeconds: 30
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: oj-judge-service
  namespace: oj-prod
spec:
  selector:
    app: oj-judge-service
  ports:
  - protocol: TCP
    port: 8083
    targetPort: 8083
  type: ClusterIP
```

**HPA 自动扩缩容**
```yaml
# k8s/oj-judge-service-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: oj-judge-service-hpa
  namespace: oj-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: oj-judge-service
  minReplicas: 5
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 12.4 CI/CD 流程

```yaml
# .github/workflows/deploy.yml
name: Deploy OJ System

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Build with Maven
        run: mvn clean package -DskipTests

      - name: Build Docker image
        run: |
          docker build -t registry.oj.com/oj-judge-service:${{ github.sha }} ./oj-judge-service
          docker tag registry.oj.com/oj-judge-service:${{ github.sha }} registry.oj.com/oj-judge-service:latest

      - name: Push to Registry
        run: |
          docker login registry.oj.com -u ${{ secrets.REGISTRY_USER }} -p ${{ secrets.REGISTRY_PASSWORD }}
          docker push registry.oj.com/oj-judge-service:${{ github.sha }}
          docker push registry.oj.com/oj-judge-service:latest

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/oj-judge-service \
            oj-judge-service=registry.oj.com/oj-judge-service:${{ github.sha }} \
            -n oj-prod
```

---

## 13. 扩展性与优化

### 13.1 性能优化清单

| 优化项 | 优化前 | 优化后 | 提升 |
|-------|-------|-------|------|
| **数据库查询** | 全表扫描 | 索引优化 | 10x |
| **接口响应时间** | 2000ms | 200ms | 10x |
| **并发处理能力** | 100 QPS | 1000 QPS | 10x |
| **缓存命中率** | 0% | 90% | - |
| **容器创建时间** | 3s | 500ms (容器池) | 6x |

### 13.2 未来扩展方向

1. **AI 辅助判题**
   - 智能错误提示
   - 代码优化建议
   - 相似题目推荐

2. **竞赛模式**
   - 周赛、双周赛
   - 实时排名
   - 虚拟竞赛

3. **社交功能**
   - 题解分享
   - 讨论区
   - 代码评论

4. **多语言支持**
   - Kotlin、Swift、Ruby
   - SQL 题目
   - Shell 脚本

5. **企业版功能**
   - 私有题库
   - 面试模拟
   - 团队协作

---

## 14. 参考现有项目的改进点

基于对 runcode 项目的分析，以下是本设计的改进：

| 方面 | runcode 现状 | 本设计改进 |
|-----|-------------|-----------|
| **并发控制** | 无队列，直接处理 | RabbitMQ 消息队列 + 动态消费者 |
| **资源限制** | 仅超时，无 CPU/内存限制 | 完整的资源配额管理 |
| **容器管理** | 每次创建销毁 | 容器对象池 + 复用 |
| **数据持久化** | 仅 localStorage | MySQL + Redis + 分库分表 |
| **性能监控** | 无 | Prometheus + Grafana 完整监控 |
| **执行监控** | 仅超时检测 | 实时资源监控 (CPU、内存) |
| **判题策略** | 硬编码 | 策略模式 + 可扩展 SPJ |
| **安全机制** | 基础隔离 | 多层防护 + 代码检查 |
| **用户系统** | 无 | 完整的用户认证授权 |
| **题目管理** | 简单文件 | 数据库管理 + 标签系统 |
| **高可用** | 单机 | 微服务 + K8s + 多副本 |
| **缓存** | 无 | 多级缓存 (Caffeine + Redis) |

---

## 总结

本设计方案基于 Java 技术栈，参考 runcode 项目的核心理念，设计了一个企业级的 LeetCode 风格在线判题系统。

**核心特点**：
1. ✅ **微服务架构**: 高内聚、低耦合、易扩展
2. ✅ **高并发支持**: 消息队列 + 容器池 + 限流
3. ✅ **完整监控**: 性能指标 + 资源监控 + 告警
4. ✅ **安全隔离**: 多层防护 + Docker 沙箱
5. ✅ **生产级部署**: K8s + CI/CD + 自动扩缩容

**技术亮点**：
- 🔥 Docker 容器对象池，执行效率提升 6 倍
- 🔥 RabbitMQ 异步判题，支持 1000+ QPS
- 🔥 多级缓存策略，缓存命中率 90%+
- 🔥 完整的资源限制，防止恶意代码
- 🔥 Prometheus 监控，实时掌控系统状态

本方案已覆盖从需求分析、架构设计、核心实现到部署运维的完整生命周期，可直接用于生产环境开发。
