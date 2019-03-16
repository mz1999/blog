# vs code 快捷键

## 界面概览

| 快捷键 | 描述 |
| --- | --- |
| cmd + shift + e | 文件资源管理器 |
| cmd + shift + f | 跨文件搜索 |
| ctrl + shift + g | 源代码管理 |
| cmd + shift + d | 启动和调试 |
| cmd + shift + x | 扩展管理 |
| cmd + shift + p | 查找并运行所有命令 |
| cmd + j | 打开、关闭panel |

## 命令行的使用

| 命令 | 描述 |
| --- | --- |
| code $path | 新窗口中打开这个文件或文件夹 |
| code -r $path | 窗口复用打开文件 |
| code -r -g $file:lineno | 打开文件，跳转到指定行 |
|  code -r -d $file1 $file2 | 比较两个文件 |
| ls | code - | 接收管道中的数据，在窗口中展示  |

## 光标移动

| 快捷键 | 描述 |
| --- | --- |
| option + 左/右方向键 | 针对单词的光标移动 |
| cmd + 左/右方向键 | 移动到行首、行尾 |
| cmd + shift + \ | 在花括号之间跳转 | 
| cmd + 上/下方向键 | 移动到文档的第一行、最后一行 |

## 文本选择

shift + 光标移动

## 删除操作

可以先选择，再删除

| 快捷键 | 描述 |
| --- | --- |
| cmd + fn + del | 删除到行尾 |
| cmd + del | 删除到行首 |
| option + del | 向前删除单词 |
| option + fn + del | 向后删除单词 |

## 代码行编辑

| 快捷键 | 描述 |
| --- | --- |
| cmd + shift + k | 删除行 |
| cmd + x | 剪切行 |
| cmd + enter | 在当前行下一行新开始一行 |
| cmd + shift + enter| 在当前行上一行新开始一行 |
| option +  上/下方向键 | 将当前行上下移动 |
| option + shift + 上/下方向键 | 将当前行上下复制 | 
| cmd + / | 将一行代码注释 |
| option + shift + a | 注释整块代码 |
| option + shift + f | 代码格式化 |
| cmd+k cmd+f  | 选中代码格式化 |
| ctrl + t | 光标前后字符调换位置 |
| cmd+shift+p transform to up/low case | 转换大小写 |
| ctrl + j | 合并代码行 |
| cmd + u | 撤销光标移动 |

## 创建多个光标

* 使用鼠标

`option + 鼠标左键`

* 使用键盘

| 快捷键 | 描述 |
| --- | --- |
| cmd + option + 上/下方向键 | 创建多个光标 |
| cmd + d | 选中相同单词，并创建多个光标 |
| option + shift+ i | 在选择的多行后创建光标 |

## 文件跳转

| 快捷键 | 描述 |
| --- | --- |
| ctrl + tab | 文件标签之间跳转 |
| cmd + p | 打开文件列表 |

## 行跳转

| 快捷键 | 描述 |
| --- | --- |
| ctrl + g | 跳转到指定行 |

## 符号跳转

| 快捷键 | 描述 |
| --- | --- |
| cmd + shift + o | 当前文件所有符号列表 |
| @: | 符号列表@后输入冒号，符号分类排列 |
| cmd + t | 在多个文件进行符号跳转 |
| cmd + F12 | 跳转到函数的实现位置 |
| shift + F12 | 函数引用列表 |
| ctrl + - | 跳回上一次光标所在位置 |
| ctrl + shift + - | 跳回下一次光标所在位置 |

## 代码自动补全

| 快捷键 | 描述 |
| --- | --- |
| ctrl+ space | 调出建议列表 |
| cmd + shift + space | 调出参数预览窗口 |
| cmd + . | 快速修复建议列表 |
| F2 | 函数名重构 |

## 代码折叠

| 快捷键 | 描述 |
| --- | --- |
| cmd+ option + [ | 最内层折叠 |
| cmd + option + ] | 最内层展开 |
| cmd+k cmd+0  | 全部折叠 |
| cmd+k cmd+j  | 全部展开 |

## 搜索
| 快捷键 | 描述 |
| --- | --- |
| cmd + f | 搜索  |
| cmd + g | 搜索，光标在编辑器内跳转 |
| cmd + option + f | 查找替换 |
| cmd + shift + f | 多文件搜索 |

## 编辑器操作

| 快捷键 | 描述 |
| --- | --- |
| cmd + \ | 拆分编辑器 |
| option + cmd + 左/右方向键 | 编辑器间切换 |
| cmd + num | 在拆分的编辑器窗口跳转 |
| Cmd +/- | 缩放整个工作区 |
| cmd + shift + p `reset zoom` | 重置缩放 |

## 专注模式

| 快捷键 | 描述 |
| --- | --- |
| cmd + b | 打开或者关闭整个视图 |
| cmd + j | 打开或者关闭面板 |
| cmd+shift+p `Toggle Zen Mode` | 切换禅模式 |
| cmd+shift+p `Toggle Centered Layout` | 切换剧中布局 |

## 命令面板

| 快捷键 | 描述 |
| --- | --- |
| cmd + shift + p | 命令面板 |

命令面板的第一个符合对应着不同的功能：

* `?` 列出所有可用功能
* `>` 用于显示所有的命令
* `@` 用于显示和跳转文件中的 “符号”（Symbols）
* `@:` 可以把符号们按类别归类
* `#` 用于显示和跳转工作区中的 “符号”（Symbols）。
* `:` 用于跳转到当前文件中的某一行。
* `edt` 显示所有已经打开的文件
* `edt active` 显示当前活动组中的文件
* `ext` 插件的管理
* `ext install` 搜索和安装插件。
* `task` 任务
* `debug` 调试功能
* `term`创建和管理终端实例
* `view` 打开各个 UI 组件

## 窗口管理

| 快捷键 | 描述 |
| --- | --- |
| ctrl + w | 窗口切换 |
| ctrl + r | 切换文件夹 |
| ctrl+r cmd+enter | 新建窗口打开文件夹 |

## 集成终端

| 快捷键 | 描述 |
| --- | --- |
| ctrl + ` | 切换集成终端 |
| ctrl + shift + ` | 新建集成终端 |
| cmd+shift+p `Run Active File In Active Terminal` | 在集成终端中运行当前脚本  |
| cmd+shift+p `Run Selected Text In Active Terminal` | 在集成终端中运行所选文本 |

## 任务管理

| 快捷键 | 描述 |
| --- | --- |
| cmd+shift+p `run task` | 自动检测当前项目中可运行的任务 |
| cmd+shift+p `Configure Task` | 配置任务 |
|  Cmd + Shift + b | 运行默认的生成任务（build task）|

## 鼠标操作

* 文本选择
    * 双击鼠标，选中单词
    * 三击鼠标，选中一行
    * 四击鼠标，选中整个文档
    * 单击行号，选中行
* 文本编辑
    * 选中后可以拖动文本到指定区域
    * 拖动过程中按`option`，变成复制文本到指定区域
* 在悬停窗口上按下`cmd`，提示函数的实现 