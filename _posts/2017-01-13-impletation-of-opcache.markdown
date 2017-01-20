---
layout: post
title: OPCache源码分析
date: 2017-01-13 10:44
comments: true
external-url:
categories: PHP-Core
---

> Opcache是PHP世界中最有名的项目之一, 它将PHP编译产生的字节码缓存到共享内存中, 在下次请求需要相同文件时, 直接从缓存读取字节码执行. 本文主要从Opcache源代码的角度, 对其主要流程进行分析介绍.

<b>本文使用的源代码版本</b>: *<a target="blank" href="https://github.com/php/php-src/tree/PHP-7.1.0/ext/opcache">PHP-7.1.0 Opcache</a>*

# 目录

* [PHP启动过程中, 如何加载扩展](#idx-topic-extension-load)

* [Opcache的对外接口](#idx-topic-entrance)

* [Opcache的内存结构(以shm为例)](#idx-topic-memory-structure)

* [Opcache启动过程的主要工作](#idx-topic-startup)

* [缓存opcode的过程](#idx-topic-opcode)

* [Opcache的清理](#idx-topic-opcache-reset)

<a id="idx-topic-extension-load" />

# PHP启动过程中, 如何加载扩展

```c
// 文件: Zend/zend_extensions.c
int zend_load_extension(const char *path);
```

这个函数定义了如何加载一个扩展(的动态链接库)到PHP的进程中, 并使两者建立关联.

其主要流程如下:

1. DL_LOAD()加载动态连接库文件(\*NIX下对应dlopen);

2. 从加载到的动态链接库中, 读取两个信息:

    * 扩展版本信息: ``` extension_version_info``` 或 ```_extension_version_info```

    * 扩展入口结构: ```zend_extension_entry``` 或 ```_zend_extension_entry```

3. 对待关联的扩展, 进行一系列检查

4. 调用```zend_register_extension()```将此扩展注册进来.

    * 这里最重要的动作, 就是将新扩展的```zend_extension_entry```对象, 加入到全局扩展列表(```ZEND_API zend_llist zend_extensions;```)中去

<a id="idx-topic-entrance" />

# Opcache的对外接口

## Opcache扩展加载入口

opcache作为一个zend_extension加载到php中.

这个角度主要暴露三个接口: ```accel_startup```, ```accel_activate```, ```accel_deactivate```.

前者用来在进程启动时, 负责opcache的共享内存初始化等工作. 后面两者, 则负责请求生命周期中, 相应的准备及清理工作.

```c
// 文件: ext/opcache/ZendAccelerator.c
ZEND_EXT_API zend_extension zend_extension_entry = {
    ACCELERATOR_PRODUCT_NAME,               /* name */
    PHP_VERSION,                            /* version */
    "Zend Technologies",                    /* author */
    "http://www.zend.com/",                 /* URL */
    "Copyright (c) 1999-2016",              /* copyright */
    accel_startup,                          /* 进程启动阶段, 对模块进行初始化 */
    NULL,                                   /* shutdown */
    accel_activate,                         /* 请求启动阶段, 激活模块 */
    accel_deactivate,                       /* 请求结束阶段, 进行资源释放 */
    NULL,                                   /* message handler */
    NULL,                                   /* op_array handler */
    NULL,                                   /* extended statement handler */
    NULL,                                   /* extended fcall begin handler */
    NULL,                                   /* extended fcall end handler */
    NULL,                                   /* op_array ctor */
    NULL,                                   /* op_array dtor */
    STANDARD_ZEND_EXTENSION_PROPERTIES
};
```

## Opcache内部accel模块

opcache自身是注册为一个zend扩展, 它在accel_startup()时, 还注册了一个zend_module.

这个zend_module主要用来对应用层(PHP)暴露API.

```c
// 文件: ext/opcache/zend_accelerator_module.c

// 函数表
static zend_function_entry accel_functions[] = {
    /* User functions */
    ZEND_FE(opcache_reset,                  arginfo_opcache_none)
        ZEND_FE(opcache_invalidate,             arginfo_opcache_invalidate)
        ZEND_FE(opcache_compile_file,           arginfo_opcache_compile_file)
        ZEND_FE(opcache_is_script_cached,       arginfo_opcache_is_script_cached)
        /* Private functions */
        ZEND_FE(opcache_get_configuration,      arginfo_opcache_none)
        ZEND_FE(opcache_get_status,             arginfo_opcache_get_status)
        ZEND_FE_END
};

// ...

// 模块定义
static zend_module_entry accel_module_entry = {
    STANDARD_MODULE_HEADER,
    ACCELERATOR_PRODUCT_NAME,
    accel_functions, // 函数表
    ZEND_MINIT(zend_accelerator),
    ZEND_MSHUTDOWN(zend_accelerator),
    NULL,
    NULL,
    zend_accel_info,
    PHP_VERSION,
    NO_MODULE_GLOBALS,
    accel_post_deactivate,
    STANDARD_MODULE_PROPERTIES_EX
};
```

## Opcache对外暴露的应用层API

```boolean opcache_reset ( void );```

清理当前Opcache已经缓存的opcode(以及内部的持久化路径解析缓存).

```boolean opcache_invalidate ( string $script [, boolean $force = FALSE ] );```

主动失效指定脚本的opcode缓存.

```boolean opcache_compile_file ( string $file );```

主动编译指定脚本. 如果启用了opcode缓存会同时做缓存.

```boolean opcache_is_script_cached ( string $file );```

检测指定脚本是否已缓存.

```array opcache_get_configuration ( void );```

获取缓存相关配置项

```array opcache_get_status ([ boolean $get_scripts = TRUE ] );```

获取opcache的内存使用, 缓存状态的状态信息.

<a id="idx-topic-memory-structure" />

# Opcache的内存结构(以shm为例)

下图是笔者分析Opcache内部几个关键宏及相关内存管理, 整理的内存结构图.

<a target="blank" href="/image/posts/2017-01-13-impletation-of-opcache-shm-memory-structure.png">
    <img width="100%" src="/image/posts/2017-01-13-impletation-of-opcache-shm-memory-structure.png" />
</a>

三个关键的宏分别为:

* ZSMMG(element): 共享内存管理区
* ZCSG(element): 共享全局变量
* ZCG(element): 进程内全局变量

如图左边黑线描述的部分, Opcache在初始化共享内存时, 首先会将ZSMMG对应的共享内存管理区变量, 在进程的本地栈空间创建, 并在堆中分配一个区域(```ZSMMG(shared_segments)```), 用来持有将要创建的各个共享内存块的指针及一些其他信息.

接着, 通过shmget()创建出所需个数及大小的共享内存块.[[点击查看shm的内存块策略](#idx-topic-shm-memory-strategy)]

在创建出实际的共享内存区后, 它会接着把ZSMMG自身, 以及```ZSMMG(shared_segments)```拷贝到共享内存中去, 使其形成上图右侧蓝线描述的结构.

最终, 三个关键的宏对应的内存如图右侧红线所示.

<a href="idx-topic-shm-memory-strategy" />

## shm共享内存的内存块分配策略

opcache支持多种共享内存模型: mmap, shm, posix, win32.

可以通过```opcache.preferred_memory_model```指令控制, 不指定则会自己选择.

下面, 主要介绍shm共享内存模型下, 内存块选择的策略.

  * 首先, ```opcache.memory_consumption```决定了共享内存初始化时, 申请的空间大小, 默认值为64, 单位MB

  * shm模块中, 预定义了内存块的最大及最小尺寸

```c
#define SEG_ALLOC_SIZE_MAX 32*1024*1024
#define SEG_ALLOC_SIZE_MIN 2*1024*1024
```

  * 在进入分配前, 先确定一次内存块大小: 以SEG_ALLOC_SIZE_MAX为初值, 以1/2的速度衰减, 直到找到一个不大于请求空间2倍大小的值.

  > 以默认的```opcache.memory_consumption = 64M```为例, 此时确认的快大小就直接为SEG_ALLOC_SIZE_MAX [32M]

  * 接着, 尝试按这个块大小分配共享内存, 如果成功则退出, 否则以1/2的速度衰减, 衰减到SEG_ALLOC_SIZE_MIN仍然无法成功分配, 则宣告失败.

  > 这里, 相当于会提前分配好一个共享内存块.

  * 经过上面步骤, 最终确认了共享内存块大小. 据此计算所需共享内存块个数.

```c
*shared_segments_count = ((requested_size - 1) / seg_allocate_size) + 1;
```

  * 顺次创建剩下的```(*shared_segments_count - 1)```个共享内存块.

  > 最后一个内存块的大小, 可能会小于其他内存块, 这里确保分配出去的共享内存大小与申请大小一致.

## opcache对共享内存的管理

opcache的共享内存, 只用于缓存自身工作需要, 以及脚本的编译结果. 因此, 并不涉及一般内存管理中的回收利用, 碎片等问题.

当一个脚本的opcode被认为不可用了. 那就设置其zend_persisitent_script.corrupted = 1. 标记失效即可.

Opcache会记录标记为corrupted的数据, 浪费了多少共享内存(ZSMMG(wasted_shared_memory)).

当```ZSMMG(wasted_shared_memory) / ZCG(accel_directives).memory_consumption >= ZCG(accel_directives).max_wasted_percentage```时, Opcache会主动重置共享内存. 进行共享内存的复用.

> ```opcache.max_wasted_percentage```默认值为5, 即当有超过5%的共享内存被认为浪费时, 就会触发opcache的重置机制

<a id="idx-topic-startup" />

# Opcache启动过程的主要工作

## Opcache的模块启动过程[进程启动加载模块时]

入口函数为```ext/opcache/ZendAccelerator.c```中的函数```static int accel_startup(zend_extension *extension);```

\*. 将PHP版本相关信息求MD5值, 设置到ZCG(system_id)中.

\*. 执行SAPI环境检查(比如cli是否启用).

\*. 分配并初始化共享内存.

\*. 初始化上一步分配的共享内存.

<a id="idx-topic-startup:hook-php" />

\*. 对PHP内部一些函数进行HOOK

```
* zend_compile_file => persistent_compile_file

* zend_stream_open_function => persistent_stream_open_function

* zend_resolve_path => persistent_zend_resolve_path

```

\*. 对PHP应用层一些函数进行HOOK

```

* chdir => ZEND_FN(accel_chdir)

* file_exists => accel_file_exists

* is_file => accel_is_file

* is_readable => accel_is_readable

```

\*. 对PHP的配置指令的HOOK

```
* include_path更新时回调: accel_include_path_on_modify
```

\*. 根据配置指令```opcache.user_blacklist_filename```, 初始化Opcache黑名单列表

<a id="idx-topic-startup:request" />

## Opcache的请求启动过程[请求开始之前]

入口函数为```ext/opcache/ZendAccelerator.c```中的函数```static void accel_activate(void);```

1. 将全局函数表中的```内部函数```拷贝一份到```accel_globals.function_table```中.

2. 如果```opcache.validate_root```指令启用, 则检查根路径"/"的inode编号, 太大(超过ulong)则禁用opcache

3. 解除共享内存写保护.(设置opcache.protect_memory才生效)

4. opcache重置逻辑的处理[[参考: Opcache的重置](#idx-topic-opcache-reset)].

5. 根据需要, 清理本地进程空间的其他缓存.

    * ```realpath_cache_clean();``` PHP内部realpath解析缓存

    * ```accel_reset_pcre_cache();``` 正则表达式本地缓存

<a id="idx-topic-opcode" />

# 缓存opcode的过程

Opcache对opcode的缓存过程, 主要通过两个对php内核的HOOK实现.

* ```persistent_zend_resolve_path``` HOOK了路径解析过程, 对脚本代码的路径解析, 做了Cache的读取检查

* ```persistent_compile_file``` HOOK了脚本编译过程. 这里对无缓存脚本, 走原始编译执行逻辑, 并将编译结果放入缓存; 对有缓存脚本, 则恢复缓存的opcode及执行环境.

[参考Opcache启动时对PHP内部的Hook](#idx-topic-startup:hook-php)

## 路径解析HOOK

```static zend_string* persistent_zend_resolve_path(const char *filename, int filename_len)```

opcache在新的persistent_zend_resolve_path()中, 如果当时的执行环境是以下情况, 会做额外的处理:

1. 待解析路径文件, 对应当前请求的入口脚本, 并且是刚进入时. (EG(current_execute_data)为空时)

2. include_once/require_once对应的脚本.

额外的处理, 主要指从Opcode缓存中检测并读取数据.

```c
zend_accel_hash_entry *bucket = zend_accel_hash_str_find_entry(&ZCSG(hash), key, key_length);
```

如上面代码, ZCSG(hash)就是脚本Opcode缓存的哈希表, key是根据文件名生成的KEY(实际就是文件名).

得到的bucket.data中, 存放的就是zend_persisitent_script的数据.

如果当前请求的脚本, 有opcode缓存, 则设置两个进程内全局变量:

```c
    ZCG(cache_opline) = EG(current_execute_data) ? EG(current_execute_data)->opline : NULL;
    ZCG(cache_persistent_script) = persistent_script;
```

上层persistent_compile_file()执行时, 会优先检查ZCG(cache_persistent_script), 有则使用.

> 另外, 这里有一点需要注意. ```opcache.revalidate_path```这个指令, 默认值为0, 如果设置为1, 表示在路径解析时, 不直接拿传入的脚本名查询ZCSG(hash), 而是先调用一次原始的路径解析函数, 得到realpath. 这种情况下, 就会多一次文件的stat操作.

## 编译过程HOOK

```zend_op_array *persistent_compile_file(zend_file_handle *file_handle, int type)```

新的persistent_compile_file中, 首先会检查是否需要执行opcode缓存相关逻辑. 比如未启用opcache, opcache正在重置, 设置了```opcache.file_cache_only```, eval代码等状态下, 将直接执行原始的编译过程.

接下来, 由于路径解析过程中, 如果拿到opcode的缓存, 会设置到ZCG(cache_persistent_script)中, 所以, 如果这里有, 则直接使用.

对于没有通过路径解析Hook拿到脚本缓存的, 同样是从ZCSG(hash)中查找缓存. (同样受```opcache.revalidate_path```指令影响)

如果此刻, 拿到了脚本缓存, 则进行一些必要的检查:

  * 脚本状态是否正确(```zend_persisitent_script.corrupted```复用其他进程或其他请求在此之前, 对时间戳验证, 一致性验证等的检查结果, 以及主动的opcache_invalidate()等)

  * 脚本文件是否有读权限(如果设置了```opcache.validate_permission```, 默认0, 官方文档未提及)

  * 脚本文件的时间戳验证(具体策略受```opcache.validate_timestamps```, ```opcache.revalidate_freq```影响)

  * 脚本文件一致性验证(```opcache.consistency_checks```指令控制)

如果此刻, 仍然没有拿到脚本缓存, 或者上述校验认为缓存不可用. 首先尝试文件缓存(如果开启```opcache.file_cache```). 否则, 进入编译和封装过程:

  * ```opcache_compile_file```函数封装了opcache中对编译过程的封装.

  * 第一步将Zend编译过程产出结果的关键数据结构Mock

```c
/* Save the original values for the op_array, function table and class table */
orig_active_op_array = CG(active_op_array);   // 操作码数组
orig_function_table = CG(function_table);     // 函数表
orig_class_table = CG(class_table);           // 类表
ZVAL_COPY_VALUE(&orig_user_error_handler, &EG(user_error_handler));

/* Override them with ours */
CG(function_table) = &ZCG(function_table);
EG(class_table) = CG(class_table) = &new_persistent_script->script.class_table;
ZVAL_UNDEF(&EG(user_error_handler));
```

  * 第二步, 调用Zend原生的编译函数, 执行编译

```c
zend_try {
    orig_compiler_options = CG(compiler_options);
    CG(compiler_options) |= ZEND_COMPILE_HANDLE_OP_ARRAY;
    CG(compiler_options) |= ZEND_COMPILE_IGNORE_INTERNAL_CLASSES;
    CG(compiler_options) |= ZEND_COMPILE_DELAYED_BINDING;
    CG(compiler_options) |= ZEND_COMPILE_NO_CONSTANT_SUBSTITUTION;
    op_array = *op_array_p = accelerator_orig_compile_file(file_handle, type);
    CG(compiler_options) = orig_compiler_options;
} zend_catch {
    op_array = NULL;
    do_bailout = 1; 
    CG(compiler_options) = orig_compiler_options;
} zend_end_try();
```

  * 第三步, 然后还原全局的编译环境.

  * 完成编译之后, opcache会将这些中间结果, 封装成一个zend_persisitent_script对象.

  * zend_persistent_script对象, 会被缓存到共享内存ZCSG(hash)中, 供后续使用.

经过上述过程, 得到了有效的zend_persisitent_script对象, 调用zend_accel_load_script进行脚本操作码的加载:

  * 将脚本的类表, 合并到全局的类哈希表中

  * 将脚本的函数表, 合并到全局的函数表中

  * 返回操作码列表oparray

<a id="idx-topic-opcache-reset" />

# Opcache的重置过程

Opcache的重置, 设计是类似事件驱动的一个模式.

真正的重置, 发生在每次请求过来, 执行opcache扩展的激活时. [[参见: Opcache的请求启动过程](#idx-topic-startup:request)]

在```accel_activate```中, 它检测ZCSG(restart_pending)状态, 这个状态在共享内存中.

当进程组中, 任何进程判断需要重置opcache, 或者直接接受到```opcache_reset()```这样的应用层命令时, 都是设置此状态, 并设置其他几个响应的参数, 说明重置原因.

重置过程中, 主要清理:

  * opcache相关的统计数据

```c
ZSMMG(memory_exhausted) = 0; 
ZCSG(hits) = 0; 
ZCSG(misses) = 0; 
ZCSG(blacklist_misses) = 0; 
ZSMMG(wasted_shared_memory) = 0; 
ZCSG(restart_pending) = 0; 
ZCSG(force_restart_time) = 0; 
```

  * opcode缓存: ZCSG(hash)

  * 将共享内存指针, 重置到未分配状态

  * realpath_cache_clean(): 清理php路径解析的进程内缓存

  * accel_reset_pcre_cache(): 清理php正则的进程内缓存
