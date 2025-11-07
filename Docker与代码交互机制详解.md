# Docker 容器与代码交互机制详解

## 问题场景

在线判题系统中，当我们在 Docker 容器中运行用户代码时：
- **问题 1**：如何知道代码已经运行成功并等待输入？
- **问题 2**：Java 代码如何与 Docker 容器进行输入输出交互？
- **问题 3**：如果程序在等待 `Scanner.nextLine()`，怎么动态传入测试用例？

---

## 核心概念

### 误区：认为需要"先运行，再输入"

很多人以为流程是这样的：

```
❌ 错误理解：
1. 启动容器，运行代码
2. 代码执行到 Scanner.nextLine()，阻塞等待输入
3. Java 检测到程序在等待输入
4. Java 动态传入测试用例
5. 程序继续执行，输出结果
```

**实际上，这样做极其复杂且不必要！**

### 正确方式：预先准备输入

```
✅ 正确做法：
1. 在启动容器前，先把测试用例输入写入文件或管道
2. 启动容器，运行代码，同时重定向 stdin 到输入文件
3. 程序读取输入，执行，输出结果
4. Java 读取容器的 stdout，获取输出
```

**关键点：程序不知道输入来自哪里，它只管从 stdin 读取！**

---

## 方案对比

### 方案 A：预先写入输入文件（推荐）⭐

这是 **99% 的在线判题系统** 使用的方案。

#### 工作原理

```bash
# 在容器内执行的完整命令
cat > input.txt << 'EOF'
5
1 2 3 4 5
EOF

java Main < input.txt
```

**流程图：**
```
┌─────────────────────────────────────────────────────────┐
│  Java 应用 (宿主机)                                      │
├─────────────────────────────────────────────────────────┤
│  1. 准备测试用例输入                                     │
│     String input = "5\n1 2 3 4 5\n";                   │
│                                                         │
│  2. 构建 bash 脚本                                       │
│     String script = "cat > input.txt << 'EOF'\n" +     │
│                     input + "\nEOF\n" +                 │
│                     "javac Main.java && " +             │
│                     "java Main < input.txt";            │
│                                                         │
│  3. 创建并启动容器                                       │
│     Container container = dockerClient                  │
│         .createContainerCmd(image)                      │
│         .withCmd("bash", "-c", script)                  │
│         .exec();                                        │
│     dockerClient.startContainerCmd(container.getId())   │
│         .exec();                                        │
│                                                         │
│  4. 等待容器执行完成                                     │
│     dockerClient.waitContainerCmd(container.getId())    │
│         .exec(callback);                                │
│                                                         │
│  5. 读取容器输出                                         │
│     String output = getContainerLogs(container.getId());│
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  Docker 容器                                             │
├─────────────────────────────────────────────────────────┤
│  $ bash -c "..."                                        │
│                                                         │
│  1. 写入 input.txt                                      │
│     5                                                   │
│     1 2 3 4 5                                           │
│                                                         │
│  2. 编译代码                                            │
│     $ javac Main.java                                   │
│                                                         │
│  3. 运行代码（重定向 stdin）                            │
│     $ java Main < input.txt                             │
│     ┌─────────────────────────────────┐                │
│     │  用户代码 Main.java              │                │
│     │  Scanner sc = new Scanner(       │                │
│     │      System.in); ←──────────────┼── input.txt    │
│     │  int n = sc.nextInt(); // 5     │                │
│     │  ...                             │                │
│     │  System.out.println(result);     │                │
│     └─────────────────────────────────┘                │
│                      │                                  │
│                      ▼                                  │
│  4. 输出到 stdout                                       │
│     15                                                  │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
                   Java 读取 stdout
```

#### 完整代码实现

```java
@Component
@Slf4j
public class DockerJudgeExecutor {

    @Autowired
    private DockerClient dockerClient;

    /**
     * 执行单个测试用例（预先写入输入）
     */
    public ExecutionResult executeWithPrewrittenInput(
        String code,
        String language,
        String input,
        int timeLimit
    ) throws Exception {

        String containerId = null;

        try {
            // 1. 构建执行脚本
            String script = buildScript(code, language, input);

            log.debug("执行脚本:\n{}", script);

            // 2. 创建容器
            HostConfig hostConfig = HostConfig.newHostConfig()
                .withMemory(256 * 1024 * 1024L)
                .withCpuQuota(100000L)
                .withNetworkMode("none");

            CreateContainerResponse container = dockerClient
                .createContainerCmd(getImageName(language))
                .withHostConfig(hostConfig)
                .withCmd("bash", "-c", script)
                .withUser("sandbox")
                .withAttachStdout(true)
                .withAttachStderr(true)
                .exec();

            containerId = container.getId();

            // 3. 启动容器
            long startTime = System.currentTimeMillis();
            dockerClient.startContainerCmd(containerId).exec();

            // 4. 等待执行完成（带超时）
            WaitContainerResultCallback callback = new WaitContainerResultCallback();
            Integer exitCode = dockerClient.waitContainerCmd(containerId)
                .exec(callback)
                .awaitStatusCode(timeLimit, TimeUnit.MILLISECONDS);

            long executionTime = System.currentTimeMillis() - startTime;

            // 5. 检查是否超时
            if (exitCode == null) {
                dockerClient.stopContainerCmd(containerId).withTimeout(0).exec();
                throw new TimeoutException("执行超时");
            }

            // 6. 读取输出
            String output = getContainerOutput(containerId);

            // 7. 获取资源使用情况
            long memoryUsage = getMemoryUsage(containerId);

            return ExecutionResult.builder()
                .output(output)
                .exitCode(exitCode)
                .executionTime(executionTime)
                .memoryUsage(memoryUsage)
                .build();

        } finally {
            // 8. 清理容器
            if (containerId != null) {
                try {
                    dockerClient.removeContainerCmd(containerId)
                        .withForce(true)
                        .exec();
                } catch (Exception e) {
                    log.error("清理容器失败: {}", containerId, e);
                }
            }
        }
    }

    /**
     * 构建 bash 脚本
     */
    private String buildScript(String code, String language, String input) {
        StringBuilder script = new StringBuilder();

        // 1. 写入代码文件
        script.append("cat > Main.java << 'CODE_EOF'\n");
        script.append(code);
        script.append("\nCODE_EOF\n\n");

        // 2. 写入输入文件
        if (StringUtils.isNotBlank(input)) {
            script.append("cat > input.txt << 'INPUT_EOF'\n");
            script.append(input);
            script.append("\nINPUT_EOF\n\n");
        }

        // 3. 编译 + 运行
        if ("java".equalsIgnoreCase(language)) {
            script.append("javac Main.java\n");
            script.append("if [ $? -ne 0 ]; then exit 1; fi\n");  // 编译失败退出
            script.append("java Main < input.txt\n");
        } else if ("python".equalsIgnoreCase(language)) {
            script.append("python3 code.py < input.txt\n");
        } else if ("cpp".equalsIgnoreCase(language)) {
            script.append("g++ -O2 -std=c++17 code.cpp -o code\n");
            script.append("if [ $? -ne 0 ]; then exit 1; fi\n");
            script.append("./code < input.txt\n");
        }

        return script.toString();
    }

    /**
     * 获取容器输出
     */
    private String getContainerOutput(String containerId) throws Exception {
        LogContainerResultCallback callback = new LogContainerResultCallback();

        dockerClient.logContainerCmd(containerId)
            .withStdOut(true)
            .withStdErr(true)
            .exec(callback)
            .awaitCompletion();

        return callback.toString();
    }

    /**
     * 自定义日志回调
     */
    private static class LogContainerResultCallback
        extends ResultCallbackTemplate<LogContainerResultCallback, Frame> {

        private final StringBuilder output = new StringBuilder();

        @Override
        public void onNext(Frame frame) {
            if (frame != null) {
                output.append(new String(frame.getPayload()));
            }
        }

        @Override
        public String toString() {
            return output.toString();
        }
    }

    /**
     * 获取内存使用
     */
    private long getMemoryUsage(String containerId) {
        try {
            Statistics stats = dockerClient.statsCmd(containerId)
                .withNoStream(true)
                .exec(new StatsCallback())
                .awaitStats();

            return stats.getMemoryStats().getUsage() / 1024; // Bytes -> KB
        } catch (Exception e) {
            log.warn("获取内存使用失败: {}", e.getMessage());
            return 0;
        }
    }

    private static class StatsCallback
        extends ResultCallbackTemplate<StatsCallback, Statistics> {

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

#### 使用示例

```java
@Test
public void testJudge() {
    String code =
        "import java.util.*;\n" +
        "public class Main {\n" +
        "    public static void main(String[] args) {\n" +
        "        Scanner sc = new Scanner(System.in);\n" +
        "        int n = sc.nextInt();\n" +
        "        int sum = 0;\n" +
        "        for (int i = 0; i < n; i++) {\n" +
        "            sum += sc.nextInt();\n" +
        "        }\n" +
        "        System.out.println(sum);\n" +
        "    }\n" +
        "}\n";

    String input = "5\n1 2 3 4 5\n";

    ExecutionResult result = executor.executeWithPrewrittenInput(
        code, "java", input, 5000
    );

    System.out.println("输出: " + result.getOutput());  // 15
    System.out.println("耗时: " + result.getExecutionTime() + "ms");
    System.out.println("内存: " + result.getMemoryUsage() + "KB");
}
```

**优点：**
- ✅ 简单高效
- ✅ 不需要检测程序状态
- ✅ 程序完全透明（不知道输入来自文件）
- ✅ 99% 的判题系统都用这个方案

**缺点：**
- ❌ 无法动态交互（不支持聊天机器人类的程序）

---

### 方案 B：使用 Docker exec 动态输入

如果真的需要在程序运行中途动态输入（例如交互式程序），可以使用 `docker exec`。

#### 工作原理

```
1. 创建容器并启动，运行代码
2. 代码开始执行，阻塞在 Scanner.nextLine()
3. Java 使用 docker exec 向容器的 stdin 写入数据
4. 程序读取到输入，继续执行
5. Java 读取容器的 stdout
```

**流程图：**

```
┌─────────────────────────────────────────────────────────┐
│  Java 应用                                               │
├─────────────────────────────────────────────────────────┤
│  1. 创建容器，运行代码（不阻塞）                          │
│     container = dockerClient.createContainerCmd(...)     │
│         .withCmd("java", "Main")                         │
│         .withStdinOpen(true)  ← 关键：保持 stdin 开启    │
│         .withTty(false)                                  │
│         .exec();                                         │
│     dockerClient.startContainerCmd(...)                  │
│                                                          │
│  2. 程序在容器内开始执行                                  │
│     （此时程序可能在等待输入）                            │
│                                                          │
│  3. 通过 attach 写入输入                                 │
│     dockerClient.attachContainerCmd(container.getId())   │
│         .withStdIn(inputStream)  ← 传入输入流            │
│         .withStdOut(outputStream)                        │
│         .exec(callback);                                 │
│                                                          │
│  4. 读取输出                                             │
│     String output = outputStream.toString();             │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  Docker 容器                                             │
├─────────────────────────────────────────────────────────┤
│  $ java Main                                            │
│                                                         │
│  Scanner sc = new Scanner(System.in);                   │
│  String line = sc.nextLine();  ← 阻塞等待              │
│                   ▲                                     │
│                   │ Java 通过 attach 写入               │
│                   │ "Hello World"                       │
│                   │                                     │
│  System.out.println("You said: " + line);               │
│                   │                                     │
│                   ▼                                     │
│  stdout: "You said: Hello World"                        │
└─────────────────────────────────────────────────────────┘
```

#### 代码实现

```java
/**
 * 动态输入方案（使用 attach）
 */
public class InteractiveExecutor {

    @Autowired
    private DockerClient dockerClient;

    /**
     * 交互式执行
     */
    public String executeInteractive(
        String code,
        String language,
        List<String> inputs
    ) throws Exception {

        String containerId = null;

        try {
            // 1. 写入代码到临时容器
            String tempContainerId = createTempContainer(language);
            writeCodeToContainer(tempContainerId, code, language);
            compileInContainer(tempContainerId, language);

            // 2. 创建运行容器（保持 stdin 开启）
            CreateContainerResponse container = dockerClient
                .createContainerCmd(getImageName(language))
                .withCmd("java", "Main")
                .withStdinOpen(true)   // 关键：保持 stdin 开启
                .withTty(false)
                .withAttachStdin(true)
                .withAttachStdout(true)
                .withAttachStderr(true)
                .exec();

            containerId = container.getId();

            // 3. 复制编译好的代码到运行容器
            copyBetweenContainers(tempContainerId, containerId, "/sandbox/");

            // 4. 启动容器
            dockerClient.startContainerCmd(containerId).exec();

            // 5. Attach 到容器，进行交互
            ByteArrayInputStream inputStream = new ByteArrayInputStream(
                String.join("\n", inputs).getBytes()
            );

            ByteArrayOutputStream outputStream = new ByteArrayOutputStream();

            AttachContainerResultCallback callback =
                new AttachContainerResultCallback() {
                    @Override
                    public void onNext(Frame frame) {
                        try {
                            outputStream.write(frame.getPayload());
                        } catch (IOException e) {
                            log.error("写入输出失败", e);
                        }
                    }
                };

            dockerClient.attachContainerCmd(containerId)
                .withStdIn(inputStream)
                .withStdOut(true)
                .withStdErr(true)
                .withFollowStream(true)
                .exec(callback);

            // 6. 等待完成
            callback.awaitCompletion(10, TimeUnit.SECONDS);

            // 7. 返回输出
            return outputStream.toString();

        } finally {
            if (containerId != null) {
                dockerClient.removeContainerCmd(containerId)
                    .withForce(true)
                    .exec();
            }
        }
    }
}
```

**优点：**
- ✅ 支持真正的交互式程序
- ✅ 可以根据程序输出动态决定下一次输入

**缺点：**
- ❌ 实现复杂
- ❌ 需要检测程序状态（是否在等待输入）
- ❌ 性能较差
- ❌ 容易出现死锁（程序在等待输入，但Java没有写入）

**适用场景：**
- 交互式机器人程序
- 游戏 AI 对战
- 命令行工具测试

---

### 方案 C：使用 Docker API 的 Exec + Stdin

这是方案 B 的简化版本，适合多次输入的场景。

```java
/**
 * 使用 exec 命令多次输入
 */
public List<String> executeMultipleInputs(
    String containerId,
    List<String> inputs
) throws Exception {

    List<String> outputs = new ArrayList<>();

    for (String input : inputs) {
        // 每次输入创建一个 exec 实例
        ExecCreateCmdResponse exec = dockerClient
            .execCreateCmd(containerId)
            .withCmd("bash", "-c",
                String.format("echo '%s' | java Main", input))
            .withAttachStdout(true)
            .withAttachStderr(true)
            .exec();

        // 执行并获取输出
        ByteArrayOutputStream output = new ByteArrayOutputStream();

        dockerClient.execStartCmd(exec.getId())
            .exec(new ExecStartResultCallback(output, output))
            .awaitCompletion();

        outputs.add(output.toString());
    }

    return outputs;
}
```

**问题：** 这种方式每次都重新启动程序，无法保持程序状态。

---

## 如何检测程序是否在等待输入？

### 问题

如果使用交互式方案，如何知道程序已经执行到 `Scanner.nextLine()` 并阻塞等待输入？

### 方法 1：使用超时机制（简单但不准确）

```java
// 等待一段时间，假设程序已经准备好
Thread.sleep(1000);

// 然后写入输入
writeToStdin("test input");
```

**缺点：**
- ❌ 不精确（程序可能还没启动完成）
- ❌ 浪费时间（程序可能 100ms 就准备好了）

### 方法 2：检测输出流（更可靠）

```java
/**
 * 等待程序输出特定的提示信息
 */
public void waitForPrompt(OutputStream output, String prompt)
    throws Exception {

    String currentOutput = "";
    long timeout = System.currentTimeMillis() + 5000;

    while (!currentOutput.contains(prompt)) {
        if (System.currentTimeMillis() > timeout) {
            throw new TimeoutException("等待提示超时");
        }

        currentOutput = output.toString();
        Thread.sleep(50);
    }

    // 程序已经输出提示，说明在等待输入了
}

// 使用
waitForPrompt(outputStream, "请输入数字:");
writeToStdin("42\n");
```

**适用场景：**
- 程序会输出明确的提示信息
- 例如："请输入用户名："

### 方法 3：使用 strace 监控系统调用（高级）

```java
/**
 * 在容器内使用 strace 监控程序是否在调用 read()
 */
public boolean isProgramWaitingForInput(String containerId, int pid) {

    ExecCreateCmdResponse exec = dockerClient
        .execCreateCmd(containerId)
        .withCmd("bash", "-c",
            String.format("cat /proc/%d/wchan", pid))
        .exec();

    String wchan = executeAndGetOutput(exec);

    // 如果 wchan 显示 "wait_woken" 或 "pipe_wait"，说明在等待输入
    return wchan.contains("wait") || wchan.contains("pipe");
}
```

**缺点：**
- ❌ 非常复杂
- ❌ 需要在容器内安装 strace
- ❌ 性能开销大

---

## 最佳实践总结

### 对于 99% 的判题场景：使用方案 A

```java
// 1. 预先准备所有输入
String script =
    "cat > input.txt << EOF\n" +
    testCase.getInput() + "\n" +
    "EOF\n" +
    "javac Main.java && java Main < input.txt";

// 2. 创建容器，一次性执行
Container container = dockerClient.createContainerCmd(image)
    .withCmd("bash", "-c", script)
    .exec();

dockerClient.startContainerCmd(container.getId()).exec();

// 3. 等待完成，读取输出
String output = getOutput(container.getId());
```

**原因：**
- ✅ 简单、可靠、高效
- ✅ 不需要状态检测
- ✅ 支持超时控制
- ✅ LeetCode、Codeforces、牛客网都用这个方案

### 对于交互式程序：使用方案 B

仅在以下场景使用：
- 游戏 AI 对战（需要多轮交互）
- 聊天机器人测试
- 命令行工具测试

**代价：**
- 实现复杂
- 容易出现死锁
- 性能较差

---

## 常见问题解答

### Q1: 用户代码中有 `Scanner.nextLine()`，如何知道它在等待输入？

**A:** 不需要知道！预先把输入写入文件，用 `< input.txt` 重定向 stdin。程序不知道输入来自哪里，它只管从 stdin 读取。

```bash
# 程序不知道是从键盘还是文件读取，对它来说都是 stdin
java Main < input.txt
```

### Q2: 如果测试用例有多组输入，每组输入在不同行，怎么办？

**A:** 把所有输入一次性写入文件即可。

```java
String input =
    "5\n" +           // 第一行
    "1 2 3 4 5\n" +   // 第二行
    "10\n" +          // 第三行
    "hello\n";        // 第四行

// 程序会按顺序读取
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();           // 读取 5
int a = sc.nextInt();           // 读取 1
String s = sc.next();           // 读取 "hello"
```

### Q3: 编译和运行能否分开？

**A:** 可以，而且应该分开！

```java
// 方式 1: 在同一个容器内分步执行
String script =
    "javac Main.java\n" +                    // 步骤 1: 编译
    "if [ $? -ne 0 ]; then exit 1; fi\n" +  // 检查编译是否成功
    "java Main < input.txt";                 // 步骤 2: 运行

// 方式 2: 使用两次 docker exec
// 第一次 exec: 编译
ExecCreateCmdResponse compileExec = dockerClient
    .execCreateCmd(containerId)
    .withCmd("javac", "Main.java")
    .exec();

dockerClient.execStartCmd(compileExec.getId())
    .exec(callback)
    .awaitCompletion();

// 第二次 exec: 运行
ExecCreateCmdResponse runExec = dockerClient
    .execCreateCmd(containerId)
    .withCmd("bash", "-c", "java Main < input.txt")
    .exec();

dockerClient.execStartCmd(runExec.getId())
    .exec(callback)
    .awaitCompletion();
```

### Q4: 如何获取程序的实时输出？

**A:** 使用流式回调。

```java
dockerClient.attachContainerCmd(containerId)
    .withStdOut(true)
    .withFollowStream(true)
    .exec(new ResultCallback<Frame>() {
        @Override
        public void onNext(Frame frame) {
            // 实时接收输出
            String output = new String(frame.getPayload());
            System.out.print(output);
        }
    });
```

### Q5: 为什么 runcode 项目能工作？

**A:** 它使用的就是方案 A！

```typescript
// runcode/server/src/docker/index.ts (简化)
bashCmd = `cat > code.${fileSuffix} << 'EOF' ${wrapCode}`;

if (stdin) {
  bashCmd += `cat > input.txt << EOF ${wrapStdin}`;
  bashCmd += shellWithStdin;  // 例如: "java Main < input.txt"
} else {
  bashCmd += shell;           // 例如: "java Main"
}

docker.createContainer({
  Cmd: ['bash', '-c', bashCmd],
  // ...
});
```

**关键点：在执行代码前就把输入准备好了！**

---

## 完整示例：判题流程

```java
@Service
public class JudgeService {

    @Autowired
    private DockerClient dockerClient;

    public JudgeResult judge(String code, List<TestCase> testCases) {

        // 1. 创建容器
        String containerId = createContainer("java:17");
        startContainer(containerId);

        try {
            // 2. 写入代码
            writeCodeToContainer(containerId, code);

            // 3. 编译
            CompileResult compileResult = compile(containerId);
            if (!compileResult.isSuccess()) {
                return JudgeResult.compileError(compileResult.getMessage());
            }

            // 4. 执行所有测试用例
            List<TestCaseResult> results = new ArrayList<>();

            for (TestCase testCase : testCases) {
                // 写入输入文件
                writeInputToContainer(containerId, testCase.getInput());

                // 执行（重定向 stdin）
                ExecutionResult execResult = executeWithInput(
                    containerId,
                    "java Main < input.txt",
                    testCase.getTimeLimit()
                );

                // 对比输出
                boolean passed = execResult.getOutput().trim()
                    .equals(testCase.getExpectedOutput().trim());

                results.add(TestCaseResult.builder()
                    .passed(passed)
                    .executionTime(execResult.getExecutionTime())
                    .memoryUsage(execResult.getMemoryUsage())
                    .output(execResult.getOutput())
                    .build());

                // 快速失败
                if (!passed) {
                    break;
                }
            }

            // 5. 返回结果
            return JudgeResult.fromTestCaseResults(results);

        } finally {
            // 6. 清理
            removeContainer(containerId);
        }
    }

    private ExecutionResult executeWithInput(
        String containerId,
        String command,
        int timeLimit
    ) throws Exception {

        ExecCreateCmdResponse exec = dockerClient
            .execCreateCmd(containerId)
            .withCmd("bash", "-c",
                String.format("timeout %ds %s", timeLimit / 1000, command))
            .withAttachStdout(true)
            .withAttachStderr(true)
            .exec();

        ByteArrayOutputStream output = new ByteArrayOutputStream();
        long startTime = System.currentTimeMillis();

        dockerClient.execStartCmd(exec.getId())
            .exec(new ExecStartResultCallback(output, output))
            .awaitCompletion();

        long executionTime = System.currentTimeMillis() - startTime;

        return ExecutionResult.builder()
            .output(output.toString())
            .executionTime(executionTime)
            .memoryUsage(getMemoryUsage(containerId))
            .build();
    }
}
```

---

## 总结

### 关键认知

1. **不需要检测程序是否在等待输入！**
   - 预先把输入准备好
   - 使用 shell 重定向 `< input.txt`
   - 程序透明地从 stdin 读取

2. **Docker 容器的 stdin 是在创建时就确定的**
   - 不是"先运行，再输入"
   - 而是"准备好输入，再运行"

3. **99% 的场景用方案 A（预先写入输入文件）**
   - 简单、高效、可靠
   - LeetCode、Codeforces 都用这个

4. **只有真正的交互式程序才需要方案 B（attach + stdin）**
   - 游戏 AI 对战
   - 命令行工具测试
   - 实现复杂，慎用

### 最佳实践

```java
// ✅ 推荐做法
String script =
    "cat > input.txt << 'EOF'\n" +
    testCaseInput + "\n" +
    "EOF\n" +
    "javac Main.java && java Main < input.txt";

Container container = dockerClient.createContainerCmd(image)
    .withCmd("bash", "-c", script)
    .exec();
```

**核心思想：让 Shell 处理输入重定向，Java 只管读取输出！**
