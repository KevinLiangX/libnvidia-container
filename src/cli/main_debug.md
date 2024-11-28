```c
//  C 语言主程序入口，主要功能是解析命令行参数，加载所需的动态库(libnvc),然后执行特定的功能命令
int
main(int argc, char *argv[]) // argc 表示命令行参数的数量，argv是包含参数的字符串数组
{
        struct context ctx = {.uid = (uid_t)-1, .gid = (gid_t)-1};
        /*
         * 1。 初始化context结构体
         *      - context 是一个自定义的结构体，包含一些上下文信息(如用户ID，组ID等)
         *      - 使用初始化列表将uid和gid设置为无效值-1，表示尚未初始化
         */
        int rv;

        if ((rv = load_libnvc()) != 0)
                goto fail;
        /*
         * 2。 调用load_libnvc函数：load_libnvc是一个加载libnvc（NVIDIA容器相关库）的函数，如果加载失败(返回非零值)，直接跳转到fail标签处理错误
         *      
         */

        argp_parse(&usage, argc, argv, ARGP_IN_ORDER, NULL, &ctx);
        /*
         * 3. 解析命令行参数
         *      - 使用GNU argp_parse解析命令行参数
         *          - &usage 是参数解析规则的定义
         *          - argc和argv是传入的命令行参数 ---这里就是hook传入的configure的参数
         *          - ctx 是用户定义的上下文，用于存储解析结果
         */
        rv = ctx.command->func(&ctx);
        /*
         * 4. 执行具体命令
         *      - ctx.command 是一个指定命令结构体的指针，包含要执行的功能信息
         *      - ctx.command -> func 是具体的功能函数，使用ctx 上下文信息调用它。
         */
 fail:
        free(ctx.devices);
        free(ctx.init_flags);
        free(ctx.container_flags);
        free(ctx.mig_config);
        free(ctx.mig_monitor);
        free(ctx.imex_channels);
        free(ctx.driver_opts);
        return (rv); // 返回 rv, 命令执行的结果值，0表示成功，非0通常表示错误
}
```
功能与原理
主要功能: nvidia-container-cli 是 NVIDIA 容器工具链的一部分，用于配置和管理 GPU 容器化环境。
    - 此代码的作用是：
        - 加载必要的 NVIDIA 容器库；
        - 解析用户输入的命令行参数；
        - 根据解析的命令执行指定的功能；
        - 清理上下文资源。
实现原理:
    - 动态库加载:
        - load_libnvc() 负责加载 libnvc 库，提供对 NVIDIA 容器的核心功能支持。
    - 命令解析与分发:
        - argp_parse 根据定义好的参数解析规则，将用户输入的命令映射到 ctx.command，然后调用对应的功能函数。
    - 上下文驱动:
        - context 是程序运行的核心，存储了与命令执行相关的所有信息（如设备列表、容器标志、驱动选项等）。
    - 错误处理与资源管理:
        - 代码通过 fail 标签统一处理错误和资源释放，确保内存安全和资源的正确回收。
实际用途:
    - 该程序可能用于容器中检测和管理 NVIDIA GPU 的资源（如显卡设备、驱动配置等）。
    - 典型用例包括：
        - 显示 GPU 信息；
        - 配置 MIG（多实例 GPU）模式；
        - 为容器分配 GPU 资源；
        - 验证容器环境中的 GPU 驱动。
