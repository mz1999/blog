# 使用Ftrace开始内核探索之旅

操作系统内核对应用开发工程师来说就像一个黑盒，似乎很难窥探到其内部的运行机制。其实Linux内核很早就内置了一个强大的tracing工具：[Ftrace](https://www.kernel.org/doc/html/latest/trace/ftrace.html)，它几乎可以跟踪内核的所有函数，不仅可以用于调试和分析，还可以用于观察学习Linux内核的内部运行。虽然`Ftrace`在2008年就[加入了内核](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=16444a8a40d4c7b4f6de34af0cae1f76a4f6c901)，但很多应用开发工程师仍然不知道它的存在。本文就给你介绍一下`Ftrace`的基本使用。

