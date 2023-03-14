# Nginx的模块化设计"

高度模块化的设计是 Nginx 的架构基础。Nginx主框架中只提供了少量的核心代码，大量强大的功能是在各模块中实现的。

## 模块数据结构

### ngx_module_t 结构

Nginx 的模块化架构最基本的数据结构为 `ngx_module_t`，所有的模块都遵循着同样的接口设计规范。

`ngx_module_t` 是 `ngx_module_s` 的别名，定义在 `src/core/ngx_core.h` 中：

```c
typedef struct ngx_module_s          ngx_module_t;
```

而 `ngx_module_s` 在 `src/core/ngx_module.h` 中定义：

```c
struct ngx_module_s {
    ngx_uint_t            ctx_index;                        // 模块在同类型模块数组中的索引序号
    ngx_uint_t            index;                            // 模块在所有模块数组中的索引序号
    char                 *name;                             // 模块的名称
    ngx_uint_t            spare0;                           // 保留变量
    ngx_uint_t            spare1;                           // 保留变量
    ngx_uint_t            version;                          // 模块的版本号 目前只有一种，默认为1
    const char           *signature;
    void                 *ctx;                              // 模块的上下文 不同的模块指向不同的上下文
    ngx_command_t        *commands;                         // 模块的命令集，指向一个 ngx_command_t 结构数组
    ngx_uint_t            type;                             // 模块类型 
    ngx_int_t           (*init_master)(ngx_log_t *log);     // master进程启动时回调
    ngx_int_t           (*init_module)(ngx_cycle_t *cycle); // 初始化模块时回调
    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);// worker进程启动时回调
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle); // 线程启动时回调（nginx暂时无多线程模式）
    void                (*exit_thread)(ngx_cycle_t *cycle); // 线程退出时回调
    void                (*exit_process)(ngx_cycle_t *cycle);// worker进程退出时回调
    void                (*exit_master)(ngx_cycle_t *cycle); // master进程退出时回调
    uintptr_t             spare_hook0;                      // 保留字段
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};
```

`ngx_module_t`定义了`init_master`，`init_module`，`init_process`，`init_thread`，`exit_thread`，`exit_process`，`exit_master` 这7个回调方法，分别在初始化 master、初始化模块、初始化 worker 进程、初始化线程、退出线程、退出 worker 进程、退出 master 时被调用。事实上，`init_master`、`init_thread`、`exit_thread` 这3个方法目前都没有使用。

### ngx_command_t 结构

`ngx_command_t` 类型的 `commands` 数组指定了模块处理配置项的方法，在解析时配置文件会查找该表。`ngx_command_t` 是 `ngx_command_s`的别名，定义在 `src/core/ngx_core.h` 中：

```c
typedef struct ngx_command_s         ngx_command_t;
```

而 `ngx_command_s` 在 `src/core/ngx_conf_file.h` 中定义：

```c
struct ngx_command_s {
    // 配置项的名称
    ngx_str_t             name;
    // 配置项的类型
    ngx_uint_t            type;
    // 配置项解析处理函数
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    // 用于指示配置项所处内存的相对偏移位置
    ngx_uint_t            conf;
    // 表示当前配置项在整个存储配置项的结构体中的偏移位置
    ngx_uint_t            offset;
    // 配置项读取后的处理方法 必须是ngx_conf_post_t结构体的指针
    void                 *post;
};
```

## 模块类型

Nginx 模块共有6种类型，由 `ngx_module_t->type` 字段表示。类型常量分别定义在：

* src/core/ngx_conf_file.h

```c
#define NGX_CORE_MODULE      0x45524F43  /* "CORE" */
#define NGX_CONF_MODULE      0x464E4F43  /* "CONF" */
```

* src/event/ngx_event.h

```c
#define NGX_EVENT_MODULE      0x544E5645  /* "EVNT" */
```

* src/http/ngx_http_config.h

```c
#define NGX_HTTP_MODULE           0x50545448   /* "HTTP" */
```

* src/stream/ngx_stream.h

```c
#define NGX_STREAM_MODULE       0x4d525453     /* "STRM" */
```

* src/mail/ngx_mail.h

```c
#define NGX_MAIL_MODULE         0x4C49414D     /* "MAIL" */
```

所有模块间分层次、分类别的。Nginx 官方共有6大类型的模块：核心模块、配置模块、事件模块、HTTP模块、mail模块和stream模块。

![nginx modules](https://cdn.mazhen.tech/images/202211221511296.jpg)

`ngx_module_t` 中有一个类型为 `void*` 的 `ctx`成员，其定义了该模块的公共接口，每类模块都有各自特有的属性，通过 `void*` 类型的`ctx` 变量进行抽象，同类型的模块遵循同一套通用性接口。

![ngx_module_t](https://cdn.mazhen.tech/images/202211221435312.jpg)

模块都具备相同的 `ngx_module_t` 接口，但 `ctx` 指向不同的结构。由于配置类型 `NGX_CONF_MODULE`的模块只拥有1个模块 `ngx_conf_module`，所以没有具体化 `ctx`上下文成员。

配置模块和核心模块这两种模块类型是由Nginx的框架代码所定义的。

配置模块 `ngx_conf_module` 是所有模块的基础，它实现了最基本的配置项解析功能（解析 nginx.conf文件），其他模块在生效前都需要依赖配置模块处理配置指令并完成各自的准备工作。

核心模块的模块类型是 `NGX_CORE_MODULE`。目前官方的核心类型模块中共有6个具体模块，分别是 `ngx_core_module`、`ngx_errlog_module`、`ngx_events_module`、`ngx_openssl_module`、`ngx_http_module` 和 `ngx_mail_module`。Nginx 框架代码只关注 6个核心模块，而大部分模块都是非核心模块。

核心模块的 `ctx` 变量指向的是名为 `ngx_core_module_t` （src/core/ngx_module.h）的结构体。这个结构体很简单，除了一个 `name` 成员就只有 `create_conf`和 `init_conf`两个方法。

```c
typedef struct {
    ngx_str_t             name;
    // 解析配置项前Nginx框架会调用
    void               *(*create_conf)(ngx_cycle_t *cycle);
    // 解析配置项完成后，Nginx框架会调用
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```

`ngx_core_module_t` 是以配置项的解析作为基础的。 `create_conf` 回调方法来创建存储配置项的数据结构，`init_conf`回调方法使用解析出的配置项初始化核心模块功能。例如核心模块`ngx_core_module` （src/core/nginx.c）的 `ctx` 实例化为 `ngx_core_module_ctx`，定义了 `ngx_core_module_create_conf` 和 `ngx_core_module_init_conf` 回调方法。

```c
static ngx_core_module_t  ngx_core_module_ctx = {
    ngx_string("core"),
    ngx_core_module_create_conf,
    ngx_core_module_init_conf
};
```

核心模块可以定义全新的模块类型。例如核心模块 `ngx_http_module`（src/http/ngx_http.c）定义了 `NGX_HTTP_MODULE` 模块类型，所有HTTP类型的模块都由 `ngx_http_module`核心模块管理。同样的，`ngx_events_module` 定义了 `NGX_EVENT_MODULE` 模块类型，`ngx_mail_module` 定义了 `NGX_MAIL_MODULE` 模块类型。

核心模块 `ngx_http_module` 作为所有 HTTP 模块的 “代言”，负责加载所有的 HTTP 模块。同时，在类型为`NGX_HTTP_MODULE` 的模块中，`ngx_http_core_module`（src/http/ngx_http_core_module.c）作为 HTTP 核心业务与管理功能的模块，决定了 HTTP 业务的核心逻辑，以及对于具体的请求该选用哪一个HTTP 模块处理这样的工作。

```c
static ngx_http_module_t  ngx_http_core_module_ctx = {
    ngx_http_core_preconfiguration,        /* preconfiguration */
    ngx_http_core_postconfiguration,       /* postconfiguration */

    ngx_http_core_create_main_conf,        /* create main configuration */
    ngx_http_core_init_main_conf,          /* init main configuration */

    ngx_http_core_create_srv_conf,         /* create server configuration */
    ngx_http_core_merge_srv_conf,          /* merge server configuration */

    ngx_http_core_create_loc_conf,         /* create location configuration */
    ngx_http_core_merge_loc_conf           /* merge location configuration */
};

ngx_module_t  ngx_http_core_module = {
    NGX_MODULE_V1,
    &ngx_http_core_module_ctx,             /* module context */
    ngx_http_core_commands,                /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

事件模块、mail模块和HTTP模块类似，它们都在核心模块中各有1个模块作为自己的“代言人”，并在同类模块中有1个作为核心业务与管理功能的模块。

## 模块初始化

在Nginx的编译阶段执行 `configure` 后，会在 `objs` 目录下生成 `nginx_modules.c` 源文件。这个源文件中有两个很重要的全局变量 `ngx_modules` 和 `ngx_module_names`，前者保存了 Nginx 将要使用的全部模块，后者则记录了这些模块的名称。

```c
ngx_module_t *ngx_modules[] = {
    &ngx_core_module,
    &ngx_errlog_module,
    &ngx_conf_module,
    &ngx_openssl_module,
    &ngx_regex_module,
    &ngx_events_module,
    &ngx_event_core_module,
    ...
};

char *ngx_module_names[] = {
    "ngx_core_module"
    "ngx_errlog_module",
    "ngx_conf_module",
    "ngx_openssl_module",
    "ngx_regex_module",
    "ngx_events_module",
    "ngx_event_core_module",
    ...
};
```

### main函数入口

main函数入口定义在 `src/core/nginx.c` 文件中。

```c
int ngx_cdecl
main(int argc, char *const *argv)
{
    ngx_buf_t        *b;
    ngx_log_t        *log;
    ngx_uint_t        i;
    ngx_cycle_t      *cycle, init_cycle;
    ngx_conf_dump_t  *cd;
    ngx_core_conf_t  *ccf;

    ngx_debug_init();
    ...     
}
```

### 预初始化

`ngx_preinit_module` 负责初始化 `ngx_modules` 数组所有模块的 `index` 和 `name`，计算`ngx_max_module`和 `ngx_modules_n`。

```c
int ngx_cdecl
main(int argc, char *const *argv)
{    
    ...
    if (ngx_preinit_modules() != NGX_OK) {
        return 1;
    }
    ...
}
```

### ngx_cycle_t 结构体

`ngx_cycle_t` 是 Nginx 框架最核心的一个结构体，其存储在系统运行过程中的所有信息，包括配置文件信息、模块信息、客户端连接、读写事件处理函数等信息。Nginx 围绕着`ngx_cycle_t` (`src/core/ngx_cycle.h`) 来控制进程的运行。

```c
typedef struct ngx_cycle_s ngx_cycle_t;
struct ngx_cycle_s {
    // 保存着所有模块存储配置项的结构体的指针，它首先是一个数组，每个数组成员又是一个指针，这个指针指向另一个存储着指针的数组
    void                  ****conf_ctx;
    ngx_pool_t               *pool;

    ngx_log_t                *log;
    ngx_log_t                 new_log;

    ngx_uint_t                log_use_stderr;  /* unsigned  log_use_stderr:1; */

    ngx_connection_t        **files;
    ngx_connection_t         *free_connections;
    ngx_uint_t                free_connection_n;

    // 模块数组
    ngx_module_t            **modules;
    ngx_uint_t                modules_n;
    ngx_uint_t                modules_used;    /* unsigned  modules_used:1; */

    ngx_queue_t               reusable_connections_queue;
    ngx_uint_t                reusable_connections_n;
    time_t                    connections_reuse_time;

    ngx_array_t               listening;
    ngx_array_t               paths;

    ngx_array_t               config_dump;
    ngx_rbtree_t              config_dump_rbtree;
    ngx_rbtree_node_t         config_dump_sentinel;

    ngx_list_t                open_files;
    ngx_list_t                shared_memory;

    ngx_uint_t                connection_n;
    ngx_uint_t                files_n;

    ngx_connection_t         *connections;
    ngx_event_t              *read_events;
    ngx_event_t              *write_events;

    ngx_cycle_t              *old_cycle;

    ngx_str_t                 conf_file;
    ngx_str_t                 conf_param;
    ngx_str_t                 conf_prefix;
    ngx_str_t                 prefix;
    ngx_str_t                 error_log;
    ngx_str_t                 lock_file;
    ngx_str_t                 hostname;
};
```

模块初始化仅关注 `cycle->modules` 和 `cycle->conf_ctx` 。

![modules & conf](https://cdn.mazhen.tech/images/202211221741885.png)

### ngx_init_cycle

模块的初始化是在 main 中调用函数 `ngx_init_cycle` 完成的：

```c
int ngx_cdecl
main(int argc, char *const *argv)
{    
    ...
    cycle = ngx_init_cycle(&init_cycle);
    if (cycle == NULL) {
        if (ngx_test_config) {
            ngx_log_stderr(0, "configuration file %s test failed",
                           init_cycle.conf_file.data);
        }

        return 1;
    }
    ...
}
```

函数 `ngx_init_cycle` 定义在 `src/ngx_cycle.c` 文件中。

### 创建核心模块配置解析上下文

```c
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    ...
    //配置上下文
    cycle->conf_ctx = ngx_pcalloc(pool, ngx_max_module * sizeof(void *));
    ...
    //处理core模块，cycle->conf_ctx用于存放所有CORE模块的配置
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {  //跳过非核心模块
            continue;
        }
        module = cycle->modules[i]->ctx;
        //只有ngx_core_module有create_conf回调函数,这个会调用函数会创建ngx_core_conf_t结构，
        //用于存储整个配置文件main scope范围内的信息，比如worker_processes，worker_cpu_affinity等
        if (module->create_conf) {
            rv = module->create_conf(cycle);
            if (rv == NULL) {
                ngx_destroy_pool(pool);
                return NULL;
            }
            cycle->conf_ctx[cycle->modules[i]->index] = rv;
        }
    }
    ...
}
```

### 配置文件解析

```c
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    ...
    //conf表示当前解析到的配置命令上下文，包括命令，命令参数等
    conf.args = ngx_array_create(pool, 10, sizeof(ngx_str_t));
    ...
    conf.temp_pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
    ...
    conf.ctx = cycle->conf_ctx;
    conf.cycle = cycle;
    conf.pool = pool;
    conf.log = log;
    conf.module_type = NGX_CORE_MODULE;  //conf.module_type指示将要解析这个类型模块的指令
    conf.cmd_type = NGX_MAIN_CONF;  //conf.cmd_type指示将要解析的指令的类型
    //真正开始解析配置文件中的每个命令
    if (ngx_conf_parse(&conf, &cycle->conf_file) != NGX_CONF_OK) {
     ...
    }
    ...
}
```

`cycle->conf_ctx` 是一个长度为 `ngx_max_module` 的数组，每个元素是各个模块的配置结构体。`conf_ctx` 的创建过程和组织方式会在 `ngx_conf_parse` 中完成。

`ngx_conf_parse` 解析配置文件中的命令，`conf` 存放解析配置文件的上下文信息，如 `module_type` 表示将要解析模块的类型，`cmd_type` 表示将要解析的指令的类型，`ctx`指向解析出来信息的存放地址，`args` 存放解析到的指令和参数。具体每个模块配信息的存放如下图所示，`NGX_MAIN_CONF`表示的是全局作用域对应的配置信息，`NGX_EVENT_CONF`表示的是 `EVENT`模块对应的配置信息，`NGX_HTTP_MAIN_CONF`，`NGX_HTTP_SRV_CONF`，`NGX_HTTP_LOC_CONF`表示的是HTTP模块对应的`main`、`server`和`local`域的配置信息。

![cycle->conf_ctx](https://cdn.mazhen.tech/images/202211221649543.png)

### 初始化核心模块的配置

```c
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    ...
    //初始化所有core module模块的config结构。调用ngx_core_module_t的init_conf,
    //在所有core module中，只有ngx_core_module有init_conf回调，
    //用于对ngx_core_conf_t中没有配置的字段设置默认值
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }
        module = cycle->modules[i]->ctx;
        if (module->init_conf) {
            if (module->init_conf(cycle,
                                  cycle->conf_ctx[cycle->modules[i]->index])
                == NGX_CONF_ERROR)
            {
                environ = senv;
                ngx_destroy_cycle_pools(&conf);
                return NULL;
            }
        }
    }
    ...
}
```

### 初始化各个模块

```c
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    ...
    if (ngx_init_modules(cycle) != NGX_OK) {
        /* fatal */
        exit(1);
    }
    ...
}
```

`ngx_init_modules` 定义在 `src/core/ngx_module.c`中：

```c
ngx_int_t
ngx_init_modules(ngx_cycle_t *cycle)
{
    ngx_uint_t  i;
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->init_module) {
            if (cycle->modules[i]->init_module(cycle) != NGX_OK) {
                return NGX_ERROR;
            }
        }
    }
    return NGX_OK;
}
```
