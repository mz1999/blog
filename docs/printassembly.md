# 使用 PrintAssembly 查看 JIT 编译后的汇编代码

`-XX:+PrintAssembly` 是一个 JVM 诊断参数，用于查看 JIT 编译器为 Java 方法生成的原生汇编代码。

直接查看汇编代码，让我们能绕开“黑盒”，直接审视 JIT 编译器对代码实际做了哪些优化，这有助于我们验证关于 JIT 行为的各种断言（例如指令重排、循环展开、安全点插入等），而不是依赖于社区中的一些“都市传说”或经验之谈。

当然，对于大多数 Java 开发者来说，阅读汇编代码并非日常工作，它可能看起来很复杂。但幸运的是，我们无需成为汇编专家。借助一些基础知识，特别是 JVM 在输出中自动插入的丰富注释，理解关键逻辑会比想象中容易。

本文的目标就是提供一个实战指南，带你从零开始设置并使用 PrintAssembly，并学会如何解读其输出。
## PrintAssembly 的作用是什么？

简单来说，`PrintAssembly` 的作用是**将 JIT 编译器生成的本地机器码（Machine Code）反汇编成人类可读的汇编语言（Assembly Language），并输出到控制台**。

这对于以下场景至关重要：
*   **深度性能分析**：查看是否存在低效的指令序列，或者 CPU 是否利用了特定的向量化指令（如 SSE, AVX）。
*   **理解 JIT 优化**：验证 JIT 是否进行了我们期望的优化，例如方法内联、循环展开、逃逸分析等。
*   **学习 JVM**：直观地了解 Java 字节码是如何被转换成与硬件交互的底层指令的。

## 使用步骤

要成功使用 `PrintAssembly`，主要分为三步：准备插件、安装插件、运行程序。

###  准备 `hsdis` 插件

#### `hsdis` 的作用及原理

当你启用 `-XX:+PrintAssembly` 参数后，JVM 内部的一个功能就被激活了。这个功能需要将 JIT 编译器刚刚生成的、存储在内存里的一串二进制机器码，翻译成我们能读懂的汇编指令。

为了完成这个翻译任务，JVM 的开发者们面临一个经典的选择：是自己从零开始为支持的每一种 CPU 架构（x86, ARM 等）都编写一个复杂的反汇编器，还是利用市面上已有的、非常成熟的专业工具？

答案显而易见，后者是更明智的选择。社区早已有了像 **GNU binutils**、**Capstone** 和 **LLVM** 这样强大且专业的反汇编引擎。

*   **GNU binutils**：经典且功能强大，但其 GPL 许可证使其无法被 Oracle/OpenJDK 直接集成和分发。
*   **Capstone**: 一个轻量级、跨平台、多架构的反汇编框架，采用友好的 BSD 许可证，是目前推荐的选择。
*   **LLVM**: 强大的编译器基础设施，其内部也包含反汇编功能。

但新的问题来了：JVM 如何与这些五花八门的外部工具对接呢？

为了解决这个问题，HotSpot JVM 的设计者定义了一个名为 **hsdis** **(HotSpot Disassembler**) 的标准接口。`hsdis` 本身并不复杂，它只做一件事：充当**桥梁**或**适配器**。JVM 在 JIT 编译完成后，会通过`hsdis`，将编译后的机器码，发送给后端真正的反汇编引擎，后者会将机器码转换成人类可读的汇编语言。

现在，我们可以把整个流程串起来，看看它是如何运作的：

1.  **JIT 编译完成**：你的 Java 方法变得足够“热”，JIT 编译器（如 C1 或 C2）将其编译成了一段原生机器码，存放在内存的某个位置。
2.  **`PrintAssembly` 激活翻译任务**：JVM 注意到你设置了 `-XX:+PrintAssembly`，于是它准备调用反汇编功能。
3.  **JVM 寻找并调用 `hsdis`**：JVM 在其库路径下寻找名为 `hsdis-*.so` 的文件。找到后，它通过标准接口调用 `hsdis`，并告诉它：“嘿，在这块内存地址（比如 `0x00007c1ef03a0700`），有一段特定长度的机器码，你帮我翻译一下。”
4.  **`hsdis` 转发请求给后端**：`hsdis` 插件收到请求后，自己并不会进行翻译。它只是一个“中间人”，它会立刻转身，调用它在编译时就链接好的后端引擎，并将 JVM 给它的信息原封不动地传递过去。
5.  **后端引擎执行反汇编**：Capstone 这样的专业引擎开始工作，它解析二进制码流，将其转换成一行行人类可读的汇编指令文本，比如 `add rbx, 1`。
6.  **结果返回**：Capstone 将翻译好的文本结果返回给 `hsdis`，`hsdis` 再将其返回给 JVM。
7.  **JVM 输出到控制台**：JVM 拿到汇编文本后，还会贴心地附加上它自己掌握的上下文信息（比如代码行号、安全点轮询注释等），最后将这整套完整的信息打印到你的控制台。

所以，`hsdis` 的核心作用是**解耦**，它让 JVM 无需关心底层究竟是哪个反汇编工具在工作，从而可以灵活地替换或选择不同的后端，也解决了因许可证问题（如 `binutils` 的 GPL）导致无法在官方 JDK 中直接集成的难题。

#### 构建 `hsdis` (Ubuntu + OpenJDK 21u + Capstone)

由于多数 JDK 发行版不自带 `hsdis`，我们需要自行构建。下面以 Ubuntu 平台为例，使用 Capstone 作为后端引擎，从 OpenJDK 21u 源码构建。

**a. 准备构建环境**

```bash
# 安装基础构建工具和 Capstone 开发库
sudo apt update
sudo apt install -y build-essential libcapstone-dev
```

**b. 获取 OpenJDK 源码**
```bash
# 克隆 OpenJDK 21 更新版本的源码
git clone https://github.com/openjdk/jdk21u.git
cd jdk21u
```

**c. 配置并构建**
```bash
# 运行 configure，指定使用 capstone 作为 hsdis 的后端
bash configure --with-hsdis=capstone

# 检查配置结果，确保 hsdis (capstone) 被正确识别
# ...

# 运行 make 命令构建 hsdis
make build-hsdis
```
构建成功后，你会在 `build/linux-x86_64-server-release/support/hsdis/` 目录下找到目标文件 `hsdis-amd64.so`。

### 安装 `hsdis` 插件

将构建好的 `hsdis-amd64.so` 文件复制到你正在使用的 JDK 的相应目录中，JVM 启动时会自动在该位置查找它。

```bash
# 将插件复制到 server VM 的库目录下
cp build/linux-x86_64-server-release/support/hsdis/hsdis-amd64.so ${JAVA_HOME}/lib/server/
```

### 运行程序

插件准备就绪，我们现在可以开始运行程序并查看输出了。首先准备好我们的示例代码：

**示例程序：`SafepointTest.java`**
```java
public class SafepointTest {
    static final double PI = 3.14159;
    public static void main(String[] args) {
        for (int i = 0; i < 200_000; i++) {
            countedLoop();
            nonCountedLoop();
        }
    }
    public static void countedLoop() {
        long sum = 0;
        for (int i = 0; i < 1000; i++) {
            sum += i;
        }
        double result = sum * PI;
    }
    public static void nonCountedLoop() {
        long i = 0;
        while (i < 10000) {
            i++;
        }
    }
}
```
先编译它：`javac SafepointTest.java`

#### 解锁诊断参数：`-XX:+UnlockDiagnosticVMOptions`

要让 JVM 输出 JIT 编译后的汇编代码，核心参数是 `-XX:+PrintAssembly`。不过，这个参数属于 JVM 的**诊断选项（Diagnostic Options）**。

诊断选项功能强大，但如果使用不当，可能影响程序的稳定性和性能，甚至导致 JVM 崩溃。因此，JVM 的开发者默认将这些选项“锁定”，以防被误用。要使用它们，必须先通过 `-XX:+UnlockDiagnosticVMOptions` 这个参数来“解锁”。

**需要特别注意的是，解锁参数必须放在所有诊断参数的前面。**

#### 启用汇编打印：`-XX:+PrintAssembly`

解锁之后，我们就可以使用核心参数 `-XX:+PrintAssembly`。

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly SafepointTest
```

如果你直接运行上面的命令，JVM 会打印出**所有**被 JIT 编译的方法的汇编代码，其中会包含大量来自 Java 标准库的内部方法。这会产生大量的信息，让你很难找到自己关心的部分。

####  `-XX:CompileCommand` 精确控制编译和输出

`CompileCommand` 是一个功能非常强大的诊断参数，它允许我们对 JIT 编译器的行为进行细粒度的控制。它的功能包括：

*   **`exclude`**: 禁止 JIT 编译某个方法。
*   **`inline` / `dontinline`**: 强制或禁止对某个方法进行内联。
*   **`log`**: 只将特定方法的编译日志输出到 `hotspot.log` 文件中。
*   **`option`**: 只对特定方法启用某个 JVM 诊断选项（如 `PrintInlining`）。
*   **`print`**: 与 PrintAssembly 选项类似，但可以指定方法名或通配符。

当你觉得直接使用 `-XX:+PrintAssembly` 输出的信息过多，`CompileCommand` 的 `print` 指令就派上了用场。

在我们的例子中，可以这么写：
*   `-XX:CompileCommand="print,SafepointTest::*"`

这条指令地告诉 JVM：“请只打印出 `SafepointTest` 这个类里所有方法的 JIT 汇编代码。”

`SafepointTest::*`: 是一个匹配模式，`::` 是类与方法的分隔符，`*` 是通配符，代表 `SafepointTest` 类中的**所有方法**。

通过这种方式，我们就能只关注自己编写的代码，大大提高了分析效率。

#### 设置汇编语法：`-XX:PrintAssemblyOptions=intel`

这是一个可选但很有帮助的参数。主流的 x86 汇编有两种语法格式：
*   **Intel 语法**: `指令 目标, 源` (例如 `mov rax, rbx`)
*   **AT&T 语法**: `指令 源, 目标` (例如 `movq %rbx, %rax`)

对于大多数开发者而言，Intel 语法更直观易读。通过设置此参数，我们可以让输出更符合我们的阅读习惯。

综合起来，最终的命令如下：

```bash
java -XX:+UnlockDiagnosticVMOptions \
     -XX:CompileCommand="print,SafepointTest::*" \
     -XX:PrintAssemblyOptions=intel \
     SafepointTest
```
执行它，你就能在控制台看到 `SafepointTest.java` 中几个方法被 JIT 编译后的汇编代码了。

## 解读汇编输出

你的控制台会打印出多个方法的汇编代码。一个方法可能被多次编译。JVM 为了平衡启动速度和长期运行性能，采用了**分层编译（Tiered Compilation）**的策略。

**C1 编译器编译 (Client Compiler)**：

*   C1 编译器会快速地将热点代码（被频繁调用的代码）编译成本地机器码。
*   它的编译速度快，但优化程度较低。
*   C1 编译出的代码中会**内嵌大量的性能分析探针（Profiling）**，用于收集代码运行时的信息，比如分支跳转频率、调用的具体类型等。这些信息将用于更高层次的优化。
*   输出中 `============================= C1-compiled nmethod ==============================` 块就是 C1 编译的结果。出现了两次是因为 JVM 在收集到更多信息后，用 C1 对其进行了再次编译。

**C2 编译器编译 (Server Compiler)**：

*   当一个方法变得“非常热”时（即被执行了足够多次，并且 C1 收集到了足够的性能分析数据），C2 编译器会介入。
*   C2 编译器会进行非常深入和激进的优化，例如方法内联、循环展开、死代码消除等。
*   它的编译过程更耗时，但生成的代码执行效率极高。
*   输出中 `============================= C2-compiled nmethod ==============================` 块就是 C2 编译的结果。

我们选取其中一段输出来分析其结构，以 C2 编译的 `nonCountedLoop` 方法为例：

```
============================= C2-compiled nmethod ============================== <-- 标题
----------------------------------- Assembly ----------------------------------- <-- 段落标题

Compiled method (c2) 23    7 %     4       SafepointTest::nonCountedLoop @ 2 (18 bytes)  <-- 编译摘要
 total in heap  [0x00007c1ef03a0590,0x00007c1ef03a0810] = 640 <-- 内存布局
 relocation     [0x00007c1ef03a06e8,0x00007c1ef03a0700] = 24
 main code      [0x00007c1ef03a0700,0x00007c1ef03a07a0] = 160
 ...

[Disassembly] <-- 反汇编内容开始标志
--------------------------------------------------------------------------------
[Constant Pool (empty)] <-- 常量池

--------------------------------------------------------------------------------

[Verified Entry Point] <-- 验证入口点
  # {method} {0x00007c1e7b400428} 'nonCountedLoop' '()V' in 'SafepointTest'
  0x00007c1ef03a0710:   sub   rsp, 0x18
  ...
  0x00007c1ef03a0757:   add   rbx, 1              ; ImmutableOopMap {}
                                                            ;*goto {reexecute=1 rethrow=0 return_oop=0}
                                                            ; - (reexecute) SafepointTest::nonCountedLoop@14 (line 23)
  0x00007c1ef03a075b:   test    dword ptr [r10], eax;   {poll}
  0x00007c1ef03a075e:   cmp   rbx, 0x2710
  0x00007c1ef03a0765:   jl    0x7c1ef03a0750
  ...
  0x00007c1ef03a0779:   ret   
[Exception Handler]
  ...
[Deopt Handler Code]
  ...
--------------------------------------------------------------------------------
[/Disassembly]
```

一段完整的汇编输出主要包含以下部分：

1.  **编译摘要 (`Compiled method ...`)**:
    *   `c2`: 表示由 C2（Server）编译器编译。也可能是 `c1`。
    *   `23`: 本次编译的内部 ID。
    *   `%`: 表示这是一次 OSR (On-Stack Replacement) 编译，即在循环中途进行的编译。
    *   `4`: 编译层级（Tier 4），代表 C2 编译。
    *   `SafepointTest::nonCountedLoop @ 2 (18 bytes)`: 方法名、OSR 编译的入口字节码索引、以及原始方法的字节码大小。

2.  **内存布局 (`total in heap ...`)**:
    描述这段编译好的代码（nmethod）在内存中的详细分布，包括了代码段、重定位信息、常量、元数据等。

3.  **反汇编主体 (`[Disassembly]`)**:
    *   **`[Constant Pool]`**: 如果代码中用到了常量，会在这里列出。
    *   **`[Verified Entry Point]`**: 这是编译后方法的入口点。
    *   **汇编指令**: 核心部分。每一行包含：内存地址、指令（如 `add rbx, 1`）和 JVM 添加的注释。
        *   `add rbx, 1`: 对应 Java 代码中的 `i++`。
        *   `cmp rbx, 0x2710`: 对应 `while (i < 10000)` 的条件比较（0x2710 = 10000）。
        *   `jl 0x7c1ef03a0750`: 条件跳转指令，如果小于则跳回循环开始处，形成循环。
        *   `test dword ptr [r10], eax; {poll}`: **安全点轮询 (Safepoint Poll)**。这是 JVM 在循环中插入的检查点，用于判断是否需要暂停线程执行 GC 或其他 VM 操作。`nonCountedLoop` 因为循环次数不确定，所以必须在循环体内插入安全点检查。而 `countedLoop` 是一个可数循环（Counted Loop），JVM 知道其执行次数，通常会将安全点检查放在循环外部，从而提高循环性能。
    *   **丰富的注释**: JVM 会在指令旁添加非常有价值的注释，如 `{poll}`（安全点轮询）、`{runtime_call}`（调用 JVM 运行时）、以及与源代码的映射关系 `SafepointTest::nonCountedLoop@14 (line 23)`，极大地帮助了我们理解代码。

4.  **处理器代码 (`[Exception Handler]`, `[Deopt Handler Code]`)**:
    定义了当发生异常或需要去优化（Deoptimization）时，代码应该跳转到的地址。

## 总结

`PrintAssembly` 是一个揭示 JVM JIT 编译器工作奥秘的强大工具。虽然初看之下纷繁复杂，但通过掌握其使用方法和输出结构，你可以：
*   **定位性能瓶颈**：通过分析热点方法的汇编代码。
*   **验证优化效果**：确认 JIT 是否按预期工作。
*   **深化对 JVM 的理解**：连接上层 Java 代码和底层硬件执行的桥梁。

下次当你对一段代码的性能感到困惑时，不妨卷起袖子，用 `PrintAssembly` 深入其境，看看它到底在 CPU 上是如何运行的。
