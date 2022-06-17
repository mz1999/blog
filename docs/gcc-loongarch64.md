# GCC 为龙芯 CPU的预定义宏

GCC 会为不同 CPU 架构预定义宏，如 `__x86_64__` 代表Intel 64位CPU， `__aarch64__`代表  ARM64。 网上已经有文档对 GCC 为 CPU 的预定义宏进行了[总结](https://sourceforge.net/p/predef/wiki/Architectures/)。

这些预定义的宏有什么用呢？我们在代码中可以判断出当前的 CPU 架构，那么可以针对 不同CPU的特性，进行优化实现。例如`RocksDB` 对于获取当前时间，在 x86 平台上，会用到 [Time Stamp Counter (TSC)](https://en.wikipedia.org/wiki/Time_Stamp_Counter) 寄存器，使用 `RDTSC` 指令提取 TSC 中值。对于 ARM 64 也有类似的实现：

```c
// Get the value of tokutime for right now.  We want this to be fast, so we
// expose the implementation as RDTSC.
static inline tokutime_t toku_time_now(void) {
#if defined(__x86_64__) || defined(__i386__)
  uint32_t lo, hi;
  __asm__ __volatile__("rdtsc" : "=a"(lo), "=d"(hi));
  return (uint64_t)hi << 32 | lo;
#elif defined(__aarch64__)
  uint64_t result;
  __asm __volatile__("mrs %[rt], cntvct_el0" : [ rt ] "=r"(result));
  return result;
#elif defined(__powerpc__)
  return __ppc_get_timebase();
#elif defined(__s390x__)
  uint64_t result;
  asm volatile("stckf %0" : "=Q"(result) : : "cc");
  return result;
#else
#error No timer implementation for this platform
#endif
}
```

而在将 `RocksDB` 移植到龙芯的过程中，需要修改上面的代码，判断出当前是龙芯 `loongarch64` 架构。

网上没有搜到 GCC 对龙芯 CPU 的预定宏的文档说明，只能从[源码](https://github.com/gcc-mirror/gcc/blob/master/gcc/config/loongarch/loongarch-c.cc)中找答案：

```c
void
loongarch_cpu_cpp_builtins (cpp_reader *pfile)
{
  ...
  builtin_define ("__loongarch__");
  ...
}
```

可以看到，`__loongarch__`代表龙芯CPU。 在暂时不知道龙芯是否支持`RDTSC`的情况下，只能给出通用的实现，以后再查龙芯的CPU手册进行优化。

```c
#if defined(__x86_64__) || defined(__i386__)
  ...
#elif defined(__aarch64__)
  ...
#elif defined(__powerpc__)
  ...
#elif defined(__loongarch__)
  struct timeval tv;
  gettimeofday(&tv,NULL);
  return tv.tv_sec*(uint64_t)1000000+tv.tv_usec;
#else
  #error No implementation for this platform
#endif

```
