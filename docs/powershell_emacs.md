# 为 PowerShell 配置 Emacs 模式

对于长期在 macOS 或 Linux 终端环境下工作的人来说，Windows 的终端体验可能会显得有些格格不入。尤其是 PowerShell，它的默认交互方式与熟悉的 bash 或 zsh 存在显著差异。本文将从配置 Emacs 模式入手，深入了解 Windows 终端的生态，并逐步打造一个更符合你习惯的 PowerShell 环境。

## Windows 终端：不止 cmd.exe 和 PowerShell

在 Windows 中，我们经常会接触到以下几个概念：

- **命令提示符 (cmd.exe)：** 这是 Windows 最早的命令行解释器，可以追溯到 DOS 时代。它执行简单的命令，但功能相对有限，交互体验也较为落后。我们可以将其视为 Windows 系统的“元老”，用于执行一些基本的命令操作。
- **PowerShell：** 这是一个更加现代和强大的命令行外壳，基于 .NET 框架构建，以对象而非文本处理数据，更适合自动化、脚本编写和系统管理。它可以被视为 Windows 系统的“新贵”，旨在替代 cmd.exe，成为系统管理的主力工具。
- **Windows Terminal (Windows 终端)：** 这是一个独立的应用程序，用于承载各种命令行 shell（如 cmd.exe, PowerShell, WSL 等）。它提供了标签页、分屏、自定义主题等高级特性，使得在不同外壳之间切换更加便捷。你可以将它视为一个更强大、更现代的“终端模拟器”，用于统一管理各种命令行环境。

**它们之间的关系：** **Windows Terminal** 是一个终端应用程序，它像一个容器，可以运行 **cmd.exe** 和 **PowerShell** 等命令行 shell。 **cmd.exe** 和 **PowerShell** 是不同的命令行解释器，负责解释用户输入的命令并执行操作。也就是说，**cmd.exe** 和 **PowerShell** 是 **Windows Terminal** 中的两种“引擎”。

## PSReadLine：PowerShell 的交互核心

**PSReadLine** 是 PowerShell 的一个核心模块，负责处理终端中的文本输入、历史记录、命令补全、语法高亮等关键功能。它直接影响着你在 PowerShell 中的操作效率和流畅度。如果没有 **PSReadLine**，PowerShell 的交互体验会大打折扣。

### 版本与功能

Windows 11 预装的 Windows PowerShell 5.1 虽然自带 `PSReadLine` 模块，但其功能较为基础。为了获得更流畅的体验和更强大的功能，例如预测性 IntelliSense 和更强大的多行编辑，强烈建议升级到 **PowerShell 7.2 或更高版本**。

### 升级 PowerShell (PowerShell Core)

1. **下载：** 从 [Microsoft 官方 GitHub 仓库](https://github.com/PowerShell/PowerShell/releases) 下载最新版本的 PowerShell 安装包，并按照安装向导完成安装。
2. **配置终端：** 在终端应用程序 Windows Terminal 中，增加一个新的配置文件，将命令行选项设置为你最新安装的 PowerShell 可执行文件路径，例如 `"C:\Program Files\PowerShell\7\pwsh.exe"`。
3. **验证版本：** 在 PowerShell 中运行 `Get-Module PSReadLine -ListAvailable`，检查 PSReadLine 版本。如果没有任何输出或版本低于 2.0，则需要升级，详见下文的 PSReadLine 的升级方法。

### PSReadLine 的升级

如果你的 PSReadLine 版本过旧，请运行以下命令更新：

```powershell
Install-Module -Name PSReadLine -Force
```

## Emacs 模式：让 PowerShell 更顺手

在深入配置 Emacs 模式之前，我们先来了解一下 PowerShell 的 profile 文件。

### 什么是 PowerShell Profile

PowerShell profile 相当于 PowerShell 的启动脚本。每次启动 PowerShell 时，它都会自动执行 profile 文件中的命令。这使得你可以自定义 PowerShell 环境，例如设置别名、函数、环境变量，以及在本例中，配置 PSReadLine 的编辑模式等。可以把它看作是 PowerShell 的“初始化文件”。

PowerShell 支持多个 profile 文件，它们根据不同的作用域加载。最常用的 profile 文件路径通常是 `C:\Users\<用户名>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`，它针对当前用户的所有 PowerShell 会话加载。

理解了 profile 文件的作用后，我们就可以开始配置 Emacs 模式了。

### 如何在 PowerShell profile 中设置 Emacs 模式

1. **获取 profile 文件路径** 

在 PowerShell 中运行 `$PROFILE`，获取 profile 文件路径。通常这个路径类似于`C:\Users\<用户名>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`。

2. **创建配置文件**

在默认情况下，PowerShell 的 `$PROFILE` 文件可能并不存在。可以使用以下命令检查文件是否存在：

```powershell
Test-Path $PROFILE
```

如果返回 `False`，说明该文件还没有创建。可以使用以下命令来创建该文件：

```powershell
New-Item -Path $PROFILE -ItemType File -Force
```

3. **编辑 profile 文件** 

一旦文件创建完成，您可以使用任意文本编辑器打开并编辑 `$PROFILE` 文件。例如使用 `notepad` 打开配置文件：`notepad $PROFILE`

4. **添加配置**

在配置文件中，添加以下行来启用 Emacs 模式：

```powershell
set-PSReadLineOption -EditMode Emacs
```

这行代码会在每次启动 PowerShell 时自动将编辑模式设置为 Emacs。

5. **保存并重启 PowerShell**

编辑并保存 `$PROFILE` 文件后，关闭并重新启动 PowerShell。重新打开后，PowerShell 将自动加载您的配置文件，并启用 Emacs 编辑模式。

另外，`$PROFILE` 配置文件可以用于保存各种个性化设置，包括自定义函数、别名、环境变量等。在 PowerShell 7+ 中，PowerShell 使用的 `$PROFILE` 文件路径会有细微变化，确保在编辑前检查 `$PROFILE` 的具体路径。

### 其他 PSReadLine 配置

在配置了 Emacs 模式后，PSReadLine 还有许多其他有用的配置选项，可以进一步提升你的使用体验。以下是一些常用的配置示例：

- **自定义历史记录保存路径：**
    
```powershell
Set-PSReadLineOption -HistorySavePath "path/to/your/history.txt"
```
    
通过此设置，你可以将 PowerShell 的命令历史记录保存到指定的文件中。
    
- **设置历史记录的最大条数：**
    
```POWERSHELL
Set-PSReadLineOption -MaximumHistoryCount 1000
```
    
此设置可以限制 PowerShell 保存的历史记录条数，防止历史记录文件过大。
    
- **设置预测命令的来源为历史记录：**

```powershell
Set-PSReadLineOption -PredictionSource History
```

设置命令预测的来源，可以使用历史记录或插件。
    
- **查看和自定义快捷键：**

```powershell
Get-PSReadLineKeyHandler
```
    
使用此命令可以查看当前 PSReadLine 定义的快捷键。你还可以自定义快捷键，例如：

```powershell
Set-PSReadLineKeyHandler -Key Ctrl+l -Function ClearScreen
```

这个命令将 Ctrl + L 键绑定到 `ClearScreen` 函数，实现清屏的功能。

### 常用快捷键

掌握一些常用的快捷键可以大大提高在 PowerShell 中的工作效率。以下是一些常用的快捷键：

- **PowerShell 快捷键：**
    - `Ctrl + C`: 中断当前正在执行的命令。
    - `Tab`: 自动补全命令、路径或变量名。
    - `Ctrl + R`: 搜索历史命令。
    - `↑` 或 `↓`: 浏览历史命令。
    - `Ctrl + Shift + T`: 在 Windows Terminal 中打开新的 tab

- **Emacs 模式快捷键：**
    - `Ctrl + A`: 将光标移动到行首。
    - `Ctrl + E`: 将光标移动到行尾。
    - `Ctrl + B`: 光标左移一个字符。
    - `Ctrl + F`: 光标右移一个字符。
    - `Alt + B`: 将光标移动到上一个单词的开头。
    - `Alt + F`: 将光标移动到下一个单词的开头。
    -  `Alt + D`: 删除光标后的单词。
    - `Ctrl + K`: 删除光标到行尾的所有内容。
    - `Ctrl + U`: 删除光标到行首的所有内容。

通过这个设置和快捷键的掌握，您可以在 PowerShell 中体验更接近 Emacs 的编辑体验，并提高命令行的使用效率。