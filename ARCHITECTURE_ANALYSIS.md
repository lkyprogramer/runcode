# Runcode 项目架构分析报告

## 项目概述

Runcode 是一个在线代码运行编辑器，支持多种编程语言（C++, C, Java, Rust, Node.js, Go, C#, Python, PHP 等）的在线编写和运行。该项目通过 Docker 容器化技术实现了安全隔离的代码执行环境。

## 技术栈

### 前端
- **框架**: React 18 + TypeScript
- **构建工具**: Vite 3
- **UI 库**: Ant Design 5
- **样式**: Tailwind CSS (原子化 CSS)
- **状态管理**: MobX + mobx-react-lite + mobx-persist-store
- **代码编辑器**: Monaco Editor (支持语法高亮、代码提示、格式化)
- **终端模拟器**: Xterm.js (用于终端样式输出)
- **HTTP 客户端**: Axios
- **其他工具**: ahooks, lodash, dayjs, react-router-dom

### 后端
- **框架**: Koa 2 + TypeScript
- **路由控制**: routing-controllers
- **Docker 集成**: Dockerode 3.3.4
- **依赖注入**: TypeDI
- **日志管理**: log4js
- **中间件**:
  - @koa/cors (跨域支持)
  - koa-bodyparser (请求体解析)
  - @koa/multer (文件上传支持)

### 容器化
- **Docker**: 为每种语言的不同版本构建独立镜像
- **安全隔离**: 容器内使用非 root 用户运行代码
- **网络隔离**: 禁用容器网络访问

## 系统架构流程图

### 整体交互流程

\`\`\`mermaid
sequenceDiagram
    participant User as 用户浏览器
    participant Editor as Monaco Editor
    participant React as React 组件
    participant Axios as HTTP 客户端
    participant Koa as Koa 服务器
    participant Controller as CodeController
    participant Docker as Docker 模块
    participant Dockerode as Dockerode 库
    participant Container as Docker 容器

    User->>Editor: 编写代码
    User->>React: 点击运行按钮
    React->>React: 获取编辑器代码内容
    React->>React: encodeURI 编码代码
    React->>Axios: 调用 runCode API
    Axios->>Koa: POST /code/run

    Note over Axios,Koa: 请求体：{code, type, version, stdin}

    Koa->>Controller: 路由到 run 方法
    Controller->>Controller: 参数验证
    Controller->>Controller: 检查代码长度 (max 500KB)
    Controller->>Docker: 调用 docker.run()

    Docker->>Docker: 选择镜像 (type:version)
    Docker->>Docker: 构建 bash 命令
    Docker->>Dockerode: 创建容器
    Dockerode->>Container: docker.createContainer()

    Note over Container: 容器配置：<br/>- 禁用网络<br/>- 6秒超时<br/>- 非 root 用户

    Container->>Container: 启动容器
    Container->>Container: 写入代码文件
    Container->>Container: 写入输入文件 (如有)
    Container->>Container: 执行编译命令
    Container->>Container: 执行运行命令
    Container->>Container: 收集 stdout/stderr

    Docker->>Docker: 等待执行完成或超时
    Docker->>Container: 获取日志输出
    Container-->>Docker: 返回执行结果
    Docker->>Container: 删除容器
    Docker->>Docker: 格式化输出
    Docker-->>Controller: 返回结果

    Controller-->>Koa: 返回 JSON 响应
    Koa-->>Axios: HTTP 200 + 结果数据
    Axios-->>React: 解析响应
    React->>React: 更新输出状态
    React->>User: 显示执行结果

    Note over React,User: 支持 Plain 或 Terminal 显示模式
\`\`\`

### Docker 容器执行详细流程

\`\`\`mermaid
flowchart TD
    Start([接收代码执行请求]) --> ValidateParams[验证参数]
    ValidateParams --> CheckLength{检查代码长度<br/>< 500KB?}
    CheckLength -->|否| ReturnError1[返回参数太长错误]
    CheckLength -->|是| SelectImage[根据 type 和 version<br/>选择 Docker 镜像]

    SelectImage --> BuildCommand[构建 bash 执行命令]
    BuildCommand --> HasStdin{是否有<br/>stdin 输入?}

    HasStdin -->|是| BuildWithStdin[使用 shellWithStdin<br/>构建命令]
    HasStdin -->|否| BuildNoStdin[使用 shell<br/>构建命令]

    BuildWithStdin --> CreateContainer[创建 Docker 容器]
    BuildNoStdin --> CreateContainer

    CreateContainer --> ConfigContainer[配置容器参数:<br/>- NetworkDisabled: true<br/>- StopTimeout: 6<br/>- Tty: true]

    ConfigContainer --> StartContainer[启动容器]
    StartContainer --> AttachStream[附加输出流]

    AttachStream --> WaitExecution[等待执行]
    WaitExecution --> SetTimeout[设置 6 秒超时]

    SetTimeout --> WaitComplete{容器状态}

    WaitComplete -->|6秒内完成| HandleSuccess[处理成功结果]
    WaitComplete -->|超时仍运行| HandleTimeout[标记为超时]
    WaitComplete -->|执行错误| HandleError[处理错误]

    HandleSuccess --> CollectLogs[收集容器日志]
    HandleTimeout --> CollectLogs
    HandleError --> CollectLogs

    CollectLogs --> FormatOutput[格式化输出:<br/>- 限制长度<br/>- 处理特殊字符<br/>- 折叠过多行]

    FormatOutput --> RemoveContainer[删除容器]
    RemoveContainer --> ReturnResult[返回执行结果]

    ReturnError1 --> End([结束])
    ReturnResult --> End

    style Start fill:#e1f5e1
    style End fill:#ffe1e1
    style CreateContainer fill:#e1e5ff
    style RemoveContainer fill:#ffe1e1
\`\`\`

### 代码执行命令构建流程

\`\`\`mermaid
flowchart LR
    A[代码内容] --> B[decodeURI 解码]
    B --> C[添加前缀代码<br/>如有]
    C --> D[包装为 heredoc]
    D --> E[cat > code.xxx << 'EOF']

    F[stdin 内容] --> G{是否有输入?}
    G -->|是| H[decodeURI 解码]
    G -->|否| I[直接执行]
    H --> J[写入 input.txt]

    E --> K[编译命令<br/>如需要]
    K --> G

    J --> L[执行命令 < input.txt]
    I --> M[直接执行命令]

    L --> N[Bash 命令字符串]
    M --> N

    style A fill:#e1f5e1
    style F fill:#e1f5e1
    style N fill:#ffe1e1
\`\`\`

## 核心模块详解

### 1. 前端代码发送流程

#### 文件位置
- 编辑器组件: `client/src/pages/editor/index.tsx`
- 操作面板: `client/src/pages/editor/components/Operator.tsx`
- API 服务: `client/src/pages/editor/service.ts`
- HTTP 客户端: `client/src/utils/request.ts`

#### 关键实现

**Operator.tsx 中的运行逻辑** (`client/src/pages/editor/components/Operator.tsx:115-134`)

\`\`\`typescript
const handleRunCode = async () => {
  if (timesPrevent) {
    message.info(t('editor.tips.run.fast'));
    return;
  }

  if (getEditor()) {
    const code = getEditor()?.getValue() || '';
    run({
      code: encodeURI(code),        // 编码代码内容
      type: codeType,                // 语言类型
      stdin: inputRef.current,       // 标准输入
      version: codeVersion,          // 语言版本
    });
    setTimesPrevent(true);
    setTimeout(() => {
      setTimesPrevent(false);
    }, runCodeInterval);  // 2秒防抖
  }
};
\`\`\`

**API 调用** (`client/src/pages/editor/service.ts:18-20`)

\`\`\`typescript
export function runCode(params: IRunCodeRequest): Promise<IRunCodeResponse> {
  return request.post('/code/run', params);
}
\`\`\`

**HTTP 拦截器** (`client/src/utils/request.ts:21-40`)

\`\`\`typescript
instance.interceptors.response.use(
  function (response) {
    const { code, data, message: msg } = response.data;

    if (code) {
      message.error(msg);
      throw new Error();
    }

    return data;  // 直接返回数据部分
  },
  function (error) {
    message.error(error.message);
    return Promise.reject(error);
  }
);
\`\`\`

#### 功能特性
- **防抖机制**: 2 秒内只能执行一次运行操作
- **代码自动保存**: 使用 debounce 延迟保存到 localStorage
- **双显示模式**: 支持 Plain 文本和 Terminal 终端两种输出样式
- **实时状态**: 使用 ahooks 的 useRequest 管理异步状态

### 2. 后端接口处理

#### 文件位置
- 应用入口: `server/src/app.ts`
- 控制器: `server/src/controller/code.ts`
- Docker 配置: `server/src/config/docker.ts`

#### 关键实现

**服务器启动** (`server/src/app.ts:12-31`)

\`\`\`typescript
const app = createKoaServer({
  cors: true,
  controllers: [CodeController],
});

app.listen(39005, () => {
  logger.info('应用启动成功!');
});
\`\`\`

**CodeController 路由处理** (`server/src/controller/code.ts:14-56`)

\`\`\`typescript
@JsonController('/code')
export class CodeController {
  @Post('/run')
  async run(@Body() body: IRunRequest) {
    const { code, version, type, stdin = '' } = body;

    // 参数类型验证
    if (
      typeof code !== 'string' ||
      typeof type !== 'string' ||
      (stdin && typeof stdin !== 'string')
    ) {
      return {
        code: 1,
        message: '参数有误!',
      };
    }

    // 长度限制：500KB
    if (code.length > 500000 || stdin.length > 500000) {
      return {
        code: 1,
        message: '参数太长了!',
      };
    }

    let result = {};
    try {
      const output = await docker.run({
        type,
        code,
        version,
        stdin,
      });
      result = {
        code: 0,
        data: output,
      };
    } catch (error) {
      result = {
        code: 1,
        message: JSON.stringify(error),
      };
    }
    return result;
  }
}
\`\`\`

### 3. Docker 执行机制

#### 文件位置
- Docker 执行模块: `server/src/docker/index.ts`
- 镜像配置示例: `server/src/docker/*/Dockerfile`

#### 镜像配置映射

系统为每种语言定义了执行配置 (`server/src/docker/index.ts:33-106`)

\`\`\`typescript
const imageMap: Record<string, CodeDockerOption> = {
  cpp: {
    shell: 'g++ code.cpp -o code.out && ./code.out',
    shellWithStdin: 'g++ code.cpp -o code.out && ./code.out < input.txt',
    fileSuffix: FileSuffix.cpp,
  },
  nodejs: {
    shell: 'node code.js',
    shellWithStdin: 'node code.js < input.txt',
    fileSuffix: FileSuffix.nodejs,
  },
  // ... 其他语言配置
};
\`\`\`

#### 容器创建与执行

**核心执行函数** (`server/src/docker/index.ts:120-255`)

\`\`\`typescript
export async function run(params: {
  type: CodeType;
  code: string;
  stdin: string;
  version?: string;
}) {
  // 1. 选择镜像
  let image = \`\${codeEnv}:\${version}\`;

  // 2. 构建执行命令
  let bashCmd = \`cat > code.\${fileSuffix} << 'EOF' \${wrapCode}\`;

  if (stdin) {
    bashCmd += \`cat > input.txt << EOF \${wrapStdin}
    \${shellWithStdin}\`;
  } else {
    bashCmd += \`\${shell}\`;
  }

  // 3. 创建容器
  return await new Promise((resolve, reject) => {
    docker.createContainer(
      {
        Image: image,
        Cmd: ['bash', '-c', bashCmd],
        StopTimeout: 6,              // 6秒超时
        Tty: true,
        AttachStdout: true,
        NetworkDisabled: true,        // 禁用网络
      },
      function (_err, container?: Container) {
        // 4. 启动容器
        container?.start((_err) => {
          // 5. 附加输出流
          container?.attach(
            { stream: true, stdout: true, stderr: true },
            function (_err: any, stream: any) {
              stream?.pipe(process.stdout as any);
            },
          );

          // 6. 设置超时处理
          const timeoutSig = setTimeout(handleOutput, DockerRunConfig.timeout);

          // 7. 等待容器完成
          container?.wait((status) => {
            if (!status || status?.Status === DockerRunStatus.exited) {
              clearTimeout(timeoutSig);
              handleOutput();
            }
          });
        });
      },
    );
  });
}
\`\`\`

#### 输出格式化

**处理输出内容** (`server/src/docker/index.ts:257-286`)

\`\`\`typescript
function formatOutput(outputString: string): string {
  // 1. 限制输出长度（最多 8200 字符）
  if (outputString.length > 8200) {
    outputString =
      outputString.slice(0, 4000) +
      outputString.slice(outputString.length - 4000);
  }

  // 2. 处理对象和数组的 toString 问题
  if (isType('Object', 'Array')(outputString)) {
    outputString = JSON.stringify(outputString);
  }

  // 3. 限制输出行数（最多 200 行）
  let outputStringArr = outputString.split('%0A');
  if (outputStringArr.length > 200) {
    outputStringArr = outputStringArr
      .slice(0, 100)
      .concat(
        ['%0A', '...' + encodeURI('数据太多,已折叠'), '%0A'],
        outputStringArr.slice(outputStringArr.length - 100),
      );
  }

  return outputStringArr.join('%0A');
}
\`\`\`

### 4. Docker 镜像构建

#### C++ 镜像示例 (`server/src/docker/cpp/11.5/Dockerfile`)

\`\`\`dockerfile
FROM gcc:11.5

# 创建非 root 用户
RUN useradd --create-home --no-log-init --shell /bin/bash user \\
  && adduser user sudo

USER user

WORKDIR /home/user
\`\`\`

#### Node.js 镜像示例 (`server/src/docker/nodejs/18/Dockerfile`)

\`\`\`dockerfile
FROM node:18-slim

# 创建非 root 用户
RUN useradd --create-home --no-log-init --shell /bin/bash user \\
  && adduser user sudo

USER user

WORKDIR /home/user

# 预装测试框架
RUN npm config set registry https://registry.npmmirror.com \\
  && npm init -y \\
  && npm install mocha \\
  && npm install chai \\
  && npm install typescript@5.7.2
\`\`\`

#### 镜像设计原则
1. **安全性**: 使用非 root 用户运行代码
2. **最小化**: 基于 slim 或 alpine 镜像减小体积
3. **预配置**: 预装必要的工具和依赖
4. **版本隔离**: 每个语言版本独立镜像

## 支持的语言和版本

| 语言 | 默认版本 | 支持版本 | 文件后缀 |
|------|---------|---------|---------|
| C++ | 14.2 | 11.5, 14.2 | .cpp |
| C | 14.2 | 14.2 | .c |
| Java | 8 | 8, 11, 17, 20 | .java |
| Rust | 1.83.0 | 1.83.0 | .rs |
| Node.js | 18 | 16, 18, 20, 22 | .js |
| Python | 3.9.18 | 2.7.18, 3.9.18 | .py |
| PHP | 8.4 | 7.4, 8.4 | .php |
| C# | 6.12 | 6.12 | .cs |
| Go | 1.18 | 1.18, 1.20, 1.23 | .go |
| TypeScript | - | (使用 Node.js 18) | .ts |

## 安全机制

### 1. 容器隔离
- **网络隔离**: `NetworkDisabled: true` 禁用所有网络访问
- **用户隔离**: 容器内使用非 root 用户执行代码
- **文件系统隔离**: 每个容器独立的文件系统
- **临时容器**: 执行完成后立即删除容器

### 2. 资源限制
- **执行超时**: 6 秒超时机制
- **代码长度限制**: 最大 500KB
- **输出长度限制**: 最大 8200 字符
- **输出行数限制**: 最多 200 行

### 3. 端口安全
- **Docker API 端口**: 2375 端口仅允许本地访问
- **防火墙配置**: 建议配置防火墙规则禁止外部访问
- **应用端口**: Koa 服务器监听 39005 端口

### 4. 输入验证
- **参数类型检查**: 严格的类型验证
- **长度限制**: 代码和输入数据长度限制
- **编码处理**: URI 编码/解码防止注入

## 数据流转过程

### 请求数据格式

**前端发送**
\`\`\`json
{
  "code": "console.log('Hello World')",  // URI 编码后的代码
  "type": "nodejs",                      // 语言类型
  "version": "18",                       // 版本号（可选）
  "stdin": "input data"                  // 标准输入（可选）
}
\`\`\`

**后端响应 - 成功**
\`\`\`json
{
  "code": 0,
  "data": {
    "code": 0,
    "output": "Hello World",
    "time": 0
  }
}
\`\`\`

**后端响应 - 失败**
\`\`\`json
{
  "code": 1,
  "message": "错误信息"
}
\`\`\`

### 执行状态码

\`\`\`typescript
export enum RunCodeStatus {
  success = 0,    // 执行成功
  timeout = 1,    // 执行超时
  error = 2,      // 执行错误
}
\`\`\`

## 特色功能

### 1. Monaco Editor 集成
- **语法高亮**: 支持所有主流语言
- **代码提示**: IntelliSense 智能提示
- **代码格式化**:
  - C/C++/Java: 使用 clang-format WebAssembly 模块
  - 其他语言: Monaco 内置格式化
- **多主题**: 支持明暗主题切换

### 2. 代码模板
系统为每种语言提供了默认模板代码，方便快速开始

### 3. 自动保存
- 用户编辑代码后自动保存到浏览器 localStorage
- 可配置延迟时间
- 按语言类型和版本分别存储

### 4. 双输出模式
- **Plain 模式**: 纯文本显示，适合简单输出
- **Terminal 模式**: 使用 Xterm.js 模拟终端，支持 ANSI 转义序列

### 5. 标准输入支持
- 用户可以在输入面板提供 stdin 数据
- 支持多行输入
- 自动传递给执行中的程序

### 6. 国际化支持
- 使用 i18next 和 react-i18next
- 支持中英文切换

## 性能优化

### 前端优化
1. **代码分割**: Vite 自动代码分割
2. **按需加载**: Monaco Editor worker 按需加载
3. **防抖节流**:
   - 运行按钮 2 秒防抖
   - 自动保存使用 lodash debounce
4. **静态资源 CDN**: 部分资源部署到 CDN
5. **原子化 CSS**: Tailwind CSS 按需生成

### 后端优化
1. **异步处理**: 全异步 I/O
2. **流式输出**: 容器输出流直接 pipe 到 stdout
3. **及时清理**: 容器执行完立即删除
4. **连接复用**: Dockerode 连接池

### Docker 优化
1. **镜像缓存**: 使用 Docker 层缓存
2. **最小镜像**: 使用 slim/alpine 基础镜像
3. **预装依赖**: 构建时预装常用库
4. **镜像复用**: 同一镜像可被多次创建容器使用

## 部署架构

### 开发环境
\`\`\`
┌─────────────────┐         ┌─────────────────┐
│  Client (Vite)  │         │  Server (Koa)   │
│  Port: 5173     │────────▶│  Port: 39005    │
│  localhost      │  HTTP   │  localhost      │
└─────────────────┘         └────────┬────────┘
                                     │
                                     │ Dockerode
                                     ▼
                            ┌─────────────────┐
                            │  Docker Daemon  │
                            │  Port: 2375     │
                            │  or Socket      │
                            └─────────────────┘
\`\`\`

### 生产环境
\`\`\`
┌──────────┐         ┌─────────────┐         ┌──────────────┐
│  Nginx   │────────▶│   Client    │         │    Server    │
│  (静态)  │  serve  │  (静态文件) │────────▶│  (Node.js)   │
│          │         │             │   API   │  PM2 Cluster │
└──────────┘         └─────────────┘         └──────┬───────┘
                                                     │
                                                     │
                                                     ▼
                                            ┌─────────────────┐
                                            │  Docker Daemon  │
                                            │  (多容器并发)   │
                                            └─────────────────┘
\`\`\`

## 项目文件结构

\`\`\`
runcode/
├── client/                      # 前端项目
│   ├── src/
│   │   ├── components/          # 通用组件
│   │   │   ├── CodeEditorMonaco/  # Monaco 编辑器封装
│   │   │   ├── Layout/
│   │   │   └── ...
│   │   ├── pages/               # 页面组件
│   │   │   ├── editor/          # 编辑器页面
│   │   │   │   ├── components/  # Operator, Header 等
│   │   │   │   ├── service.ts   # API 服务
│   │   │   │   └── index.tsx
│   │   │   └── ...
│   │   ├── store/               # MobX 状态管理
│   │   │   ├── config/          # 编辑器配置
│   │   │   └── ui/              # UI 状态
│   │   ├── utils/               # 工具函数
│   │   │   ├── request.ts       # Axios 封装
│   │   │   ├── storage.ts       # localStorage 封装
│   │   │   └── ...
│   │   ├── hooks/               # 自定义 Hooks
│   │   └── services/            # API 服务
│   ├── package.json
│   └── vite.config.ts
│
├── server/                      # 后端项目
│   ├── src/
│   │   ├── app.ts              # 应用入口
│   │   ├── controller/         # 控制器
│   │   │   └── code.ts         # 代码执行控制器
│   │   ├── docker/             # Docker 相关
│   │   │   ├── index.ts        # Docker 执行逻辑
│   │   │   ├── cpp/            # C++ 镜像
│   │   │   ├── nodejs/         # Node.js 镜像
│   │   │   ├── java/           # Java 镜像
│   │   │   ├── python/         # Python 镜像
│   │   │   └── ...             # 其他语言镜像
│   │   ├── config/             # 配置
│   │   │   └── docker.ts       # Docker 连接配置
│   │   ├── logger/             # 日志模块
│   │   ├── middleware/         # 中间件
│   │   └── utils/              # 工具函数
│   ├── package.json
│   └── tsconfig.json
│
├── package.json                # 根目录配置
├── README.md                   # 项目文档
└── pnpm-lock.yaml             # 依赖锁定
\`\`\`

## 关键代码路径

### 前端
- 运行按钮点击: `client/src/pages/editor/components/Operator.tsx:115`
- API 调用: `client/src/pages/editor/service.ts:18`
- HTTP 客户端: `client/src/utils/request.ts`
- 编辑器组件: `client/src/components/CodeEditorMonaco`

### 后端
- 应用入口: `server/src/app.ts:29`
- 路由处理: `server/src/controller/code.ts:14`
- Docker 执行: `server/src/docker/index.ts:120`
- 输出格式化: `server/src/docker/index.ts:257`

## 开发注意事项

### 1. Docker 配置
- **端口安全**: 确保 Docker 2375 端口仅本地访问
- **连接方式**:
  - Windows/Linux: 使用 TCP 端口连接
  - Mac/Linux: 可使用 Unix Socket 连接

### 2. 镜像构建
- 新增语言支持需要：
  1. 在 `server/src/docker/[language]/[version]/` 创建 Dockerfile
  2. 在 `server/src/docker/index.ts` 的 `imageMap` 添加配置
  3. 在 `defaultVersion` 添加默认版本
  4. 构建镜像: `docker build -t [language]:[version] .`

### 3. 前端开发
- 需要同时运行 Tailwind CSS watch: `pnpm build:tailwind:watch`
- 使用 pnpm 作为包管理器（项目强制要求）
- 环境变量在 `.env.dev` 和 `.env.production` 中配置

### 4. 后端开发
- 使用 nodemon 热重载: `pnpm dev`
- TypeScript 编译: `pnpm build`
- 生产部署: `pnpm deploy` (使用 PM2 集群模式)

## 潜在改进方向

### 1. 性能优化
- 实现 Docker 容器池，避免频繁创建销毁
- 使用 Redis 缓存相同代码的执行结果
- 实现 WebSocket 实时输出流

### 2. 功能增强
- 支持多文件项目
- 支持包依赖管理
- 增加代码分享功能
- 实现协作编辑

### 3. 安全加固
- 添加请求频率限制
- 实现用户认证和授权
- 增加代码内容安全检查
- 容器资源限制（CPU、内存）

### 4. 监控告警
- 添加 APM 性能监控
- 实现容器资源使用监控
- 异常日志收集和分析
- 构建监控看板

## 总结

Runcode 是一个设计良好的在线代码执行平台，主要优势包括：

1. **架构清晰**: 前后端分离，职责明确
2. **安全可靠**: 多层安全机制，容器隔离
3. **扩展性强**: 易于添加新语言支持
4. **用户体验好**: Monaco 编辑器 + 实时输出
5. **技术栈现代**: React 18 + Vite + TypeScript

通过 Docker 容器化技术，项目成功实现了多语言代码的安全隔离执行，为用户提供了一个功能完善的在线编程环境。

---

**分析日期**: 2025-11-06
**项目版本**: 1.6.0
**分析工具**: Claude Code
