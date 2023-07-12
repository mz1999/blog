
# 在macOS上编译和调试OpenJDK

## 获得源代码

首先从 Github 获取 OpenJDK的源代码

```bash
git clone https://github.com/openjdk/jdk.git
```

## 安装必要的软件

* Xcode
  * App Store 中获取
* Xcode Command Line Tools
  * 通过 `xcode-select --install` 命令安装
* GNU Autoconf
  * 使用 `brew install autoconf` 命令安装
* freetype
  * 使用 `brew install freetype` 命令安装
* boot JDK
  * 构建 JDK 需要预先存在的JDK，这被称为“boot JDK”。
  * 经验法则是，用于构建 JDK 主版本N的 boot JDK应该是主版本 N-1 的 JDK
  * 建议使用 [SDKMAN!](https://mahaoliang.tech/p/sdkman%E7%9A%84%E4%BD%BF%E7%94%A8/) 来安装维护 JDK 的多个版本

## 配置构建

通过运行 `bash configure` 命令来完成配置构建。这个脚本将检查你的系统，确保所有必要的依赖项都已经满足。如果一切顺利，该脚本将汇总build的配置、将使用的工具，以及 build 将使用的硬件资源：

```
Configuration summary:
* Name:           macosx-x86_64-server-release
* Debug level:    release
* HS debug level: product
* JVM variants:   server
* JVM features:   server: 'cds compiler1 compiler2 dtrace epsilongc g1gc jfr jni-check jvmci jvmti management parallelgc serialgc services shenandoahgc vm-structs zgc' 
* OpenJDK target: OS: macosx, CPU architecture: x86, address length: 64
* Version string: 22-internal-adhoc.mazhen.jdk (22-internal)
* Source date:    1689128166 (2023-07-12T02:16:06Z)

Tools summary:
* Boot JDK:       openjdk version "20.0.1" 2023-04-18 OpenJDK Runtime Environment Temurin-20.0.1+9 (build 20.0.1+9) OpenJDK 64-Bit Server VM Temurin-20.0.1+9 (build 20.0.1+9, mixed mode, sharing) (at /Users/mazhen/.sdkman/candidates/java/20.0.1-tem)
* Toolchain:      clang (clang/LLVM from Xcode 14.3.1)
* Sysroot:        /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX13.3.sdk
* C Compiler:     Version 14.0.3 (at /usr/bin/clang)
* C++ Compiler:   Version 14.0.3 (at /usr/bin/clang++)

Build performance summary:
* Build jobs:     12
* Memory limit:   16384 MB
```

## 构建 OpenJDK

一旦配置完成，你就可以开始构建 JDK 了。

```bash
make images
```

这个命令将开始构建过程，在完成后生成一个 JDK 的 image。

## 验证构建

新构建的 JDK 在 `./build/*/images/jdk`目录下，运行命令查看JDK版本

```bash
$ ./build/macosx-x86_64-server-release/images/jdk/bin/java -version
openjdk version "22-internal" 2024-03-19
OpenJDK Runtime Environment (build 22-internal-adhoc.mazhen.jdk)
OpenJDK 64-Bit Server VM (build 22-internal-adhoc.mazhen.jdk, mixed mode, sharing)
```

## 在VS code中调试 OpenJDK

首先在 VS code 中安装 [C++ extension for VS Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)。在 VS cod 中配置C++ 开发环境可以参考这篇文档 [Using Clang in Visual Studio Code](https://code.visualstudio.com/docs/cpp/config-clang-mac)。

使用 VS code 打开 OpenJDK的源代码，在恰当的位置设置好断点，点击右上角三角运行图标，选择“**Debug C/C++ file**”：

![vs code](https://cdn.mazhen.tech/images/202307121108030.png)

然后在弹出列表中选择“**(lldb) Launch**“：

![vs code](https://cdn.mazhen.tech/images/202307121109401.png)

第一次运行会弹出错误信息，我们选择打开 `launch.json`，创建新的 debugger 配置。点击右下角的 “**add configuration...**“，在弹出的列表中选择 "**C/C++： (lldb) Launch**"

![vs code](https://cdn.mazhen.tech/images/202307121115054.png)

VS code会自动添加缺省的配置，我们需要修改的是 program 和 args，设置为上面build好的 OpenJDK，以及准备运行的Java程序。

```json
"program": "${workspaceFolder}/build/macosx-x86_64-server-release/images/jdk/bin/java",
 "args": [
   "-cp",
     "/Users/mazhen/Documents/works/javaprojects/samples/playground/target/classes",
   "tech.mazhen.test.Main"
],
```

保存文件 `launch.json`，然后重新开始调试。可以在断点处停止，但是不能定位源代码，报错如下：

```
Could not load source 'make/src/java.base/unix/native/libnio/ch/Net.c': 'SourceRequest' not supported..
```

为了正确的找到源代码，需要在`launch.json`中配置 [sourceFileMap](https://code.visualstudio.com/docs/cpp/launch-json-reference#_sourcefilemap)，将源代码的编译时路径映射到本地源代码位置。完整的配置如下：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(lldb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/macosx-x86_64-server-release/images/jdk/bin/java",
            "args": [
                "-cp",
                "/Users/mazhen/Documents/works/javaprojects/samples/playground/target/classes",
                "com.apusic.test.Main"
            ],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "lldb",
            "sourceFileMap": {
                "make/": "${workspaceFolder}"
            },
        }
    ]
}
```

现在就可以在VS code 中正常调试OpenJDK的C++代码了。

![vs code](https://cdn.mazhen.tech/images/202307121407081.png)
