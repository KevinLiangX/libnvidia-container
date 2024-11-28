```c
// 配置容器化环境中的 GPU 设备，确保容器能够正确访问 GPU 资源，同时实现资源隔离和需求验证。它通过权限控制、设备检测和挂载实现安全性和功能性，
// 支持复杂的 GPU 配置如 MIG 的动态分配和监控。
int
configure_command(const struct context *ctx)
{
        struct nvc_context *nvc = NULL;
        struct nvc_config *nvc_cfg = NULL;
        struct nvc_driver_info *drv = NULL;
        struct nvc_device_info *dev = NULL;
        struct nvc_container *cnt = NULL;
        struct nvc_container_config *cnt_cfg = NULL;
        bool eval_reqs = true;
        struct devices devices = {0};
        struct devices mig_config_devices = {0};
        struct devices mig_monitor_devices = {0};
        struct error err = {0};
        int rv = EXIT_FAILURE;

        if (perm_set_capabilities(&err, CAP_PERMITTED, pcaps, nitems(pcaps)) < 0 ||
            perm_set_capabilities(&err, CAP_INHERITABLE, NULL, 0) < 0 ||
            perm_set_bounds(&err, bcaps, nitems(bcaps)) < 0) {
                warnx("permission error: %s", err.msg);
                return (rv);
        }
        /*
         * 1. 权限设置
         *      - 设置进程的能力权限(如允许加载模块，挂载设备等)
         *      - 确保不同阶段的功能调用拥有所需的系统权限
         *      - 若权限设置是被，直接返回错误
         *      -  perm_set_capabilities 这是具体干啥的
         */
        /* Initialize the library and container contexts. */
        int c = ctx->load_kmods ? NVC_INIT_KMODS : NVC_INIT;
        if (perm_set_capabilities(&err, CAP_EFFECTIVE, ecaps[c], ecaps_size(c)) < 0) {
                warnx("permission error: %s", err.msg);
                goto fail;
        }
        if ((nvc = libnvc.context_new()) == NULL ||
            (nvc_cfg = libnvc.config_new()) == NULL ||
            (cnt_cfg = libnvc.container_config_new(ctx->pid, ctx->rootfs)) == NULL) {
                warn("memory allocation failed");
                goto fail;
        }
       
        nvc->no_pivot = ctx->no_pivot;
        nvc_cfg->uid = ctx->uid;
        nvc_cfg->gid = ctx->gid;
        nvc_cfg->root = ctx->root;
        nvc_cfg->ldcache = ctx->ldcache;
        if (parse_imex_info(&err, ctx->imex_channels, &nvc_cfg->imex) < 0) {
                warnx("error parsing IMEX info: %s", err.msg);
                goto fail;
        }
        if (libnvc.init(nvc, nvc_cfg, ctx->init_flags) < 0) {
                warnx("initialization error: %s", libnvc.error(nvc));
                goto fail;
        }
        if (perm_set_capabilities(&err, CAP_EFFECTIVE, ecaps[NVC_CONTAINER], ecaps_size(NVC_CONTAINER)) < 0) {
                warnx("permission error: %s", err.msg);
                goto fail;
        }
        cnt_cfg->ldconfig = ctx->ldconfig;
        if ((cnt = libnvc.container_new(nvc, cnt_cfg, ctx->container_flags)) == NULL) {
                warnx("container error: %s", libnvc.error(nvc));
                goto fail;
        }
        /*
        * 2. 初始化NVIDIA容器运行时上下文
        *      - libnvc.context_new():创建运行时上下文，管理GPU设备和容器的交互
        *      - libnvc.config_new(): 初始化运行时配置，存储UID，GID等用户相关信息
        *      - libnvc.container_config_new(): 为容器创建配置对象，用于管理容器内部GPU的资源
        */
        
        
        /* Query the driver and device information. */
        if (perm_set_capabilities(&err, CAP_EFFECTIVE, ecaps[NVC_INFO], ecaps_size(NVC_INFO)) < 0) {
                warnx("permission error: %s", err.msg);
                goto fail;
        }
        if ((drv = libnvc.driver_info_new(nvc, ctx->driver_opts)) == NULL || // driver_info_new 获取CUDA Driver信息
            (dev = libnvc.device_info_new(nvc, NULL)) == NULL) { // device_info_new 获取GPU Driver信息
                warnx("detection error: %s", libnvc.error(nvc));
                goto fail;
        }
        /*
         * 3. 检查GPU设备和驱动信息
         *      - 使用NVIDA的运行时库检测系统中可用的GPU设备和驱动信息
         *      - 包括GPU数量，每个设备的详细信息，驱动支持的功能等
         */
        
        /* Allocate space for selecting GPU devices and MIG devices */
        if (new_devices(&err, dev, &devices) < 0) {
                warn("memory allocation failed: %s", err.msg);
                goto fail;
        }
        
        /* Allocate space for selecting which devices are available for MIG config */
        if (new_devices(&err, dev, &mig_config_devices) < 0) {
                warn("memory allocation failed: %s", err.msg);
                goto fail;
        }

        /* Allocate space for selecting which devices are available for MIG monitor */
        if (new_devices(&err, dev, &mig_monitor_devices) < 0) {
                warn("memory allocation failed: %s", err.msg);
                goto fail;
        }

        /* Select the visible GPU devices. 获取容器中可见的GPU列表 */
        if (dev->ngpus > 0) {
                if (select_devices(&err, ctx->devices, dev, &devices) < 0) {
                        warnx("device error: %s", err.msg);
                        goto fail;
                }
        }
        /*
        * 4. 选择可用设备
        *      - 根据上下文的设备配置，筛选出需要挂载的GPU设备
        *      - 如果配置中包含MIG,则还会进一步筛选出可用MIG配置或监控的设备
        */
        
        /* Select the devices available for MIG config among the visible devices. */
        if (select_mig_config_devices(&err, ctx->mig_config, &devices, &mig_config_devices) < 0) {
                warnx("mig-config error: %s", err.msg);
                goto fail;
        }

        /* Select the devices available for MIG monitor among the visible devices. */
        if (select_mig_monitor_devices(&err, ctx->mig_monitor, &devices, &mig_monitor_devices) < 0) {
                warnx("mig-monitor error: %s", err.msg);
                goto fail;
        }

        /*
         * Check the container requirements.
         * Try evaluating per visible device first, and globally otherwise.
         */
        for (size_t i = 0; i < devices.ngpus; ++i) {
                struct dsl_data data = {drv, devices.gpus[i]};
                for (size_t j = 0; j < ctx->nreqs; ++j) {
                        if (dsl_evaluate(&err, ctx->reqs[j], &data, rules, nitems(rules)) < 0) {
                                warnx("requirement error: %s", err.msg);
                                goto fail;
                        }
                }
                eval_reqs = false;
        }
        for (size_t i = 0; i < devices.nmigs; ++i) {
                struct dsl_data data = {drv, devices.migs[i]->parent};
                for (size_t j = 0; j < ctx->nreqs; ++j) {
                        if (dsl_evaluate(&err, ctx->reqs[j], &data, rules, nitems(rules)) < 0) {
                                warnx("requirement error: %s", err.msg);
                                goto fail;
                        }
                }
                eval_reqs = false;
        }
        if (eval_reqs) {
                struct dsl_data data = {drv, NULL};
                for (size_t j = 0; j < ctx->nreqs; ++j) {
                        if (dsl_evaluate(&err, ctx->reqs[j], &data, rules, nitems(rules)) < 0) {
                                warnx("requirement error: %s", err.msg);
                                goto fail;
                        }
                }
        }
        /* 5。 检查容器需求，是否需要特定的GPU功能或驱动支持 */
        
        /* Mount the driver, visible devices, mig-configs, mig-monitors, and imex-channels. */
        if (perm_set_capabilities(&err, CAP_EFFECTIVE, ecaps[NVC_MOUNT], ecaps_size(NVC_MOUNT)) < 0) {
                warnx("permission error: %s", err.msg);
                goto fail;
        }
        if (libnvc.driver_mount(nvc, cnt, drv) < 0) {
                warnx("mount error: %s", libnvc.error(nvc));
                goto fail;
        }
        /*
         * 6。 挂载设备和驱动
         *      - 将选定的GPu设备，驱动程序和功能(如：MIG配置)挂载到容器中
         *      - 保证容器内部能够访问和使用主机的GPU资源
         */
        for (size_t i = 0; i < devices.ngpus; ++i) {
                if (libnvc.device_mount(nvc, cnt, devices.gpus[i]) < 0) {
                        warnx("mount error: %s", libnvc.error(nvc));
                        goto fail;
                }
        }
        if (!mig_config_devices.all && !mig_monitor_devices.all) {
                for (size_t i = 0; i < devices.nmigs; ++i) {
                        if (libnvc.mig_device_access_caps_mount(nvc, cnt, devices.migs[i]) < 0) {
                                warnx("mount error: %s", libnvc.error(nvc));
                                goto fail;
                        }
                }
        }
        if (mig_config_devices.all && mig_config_devices.ngpus) {
                if (libnvc.mig_config_global_caps_mount(nvc, cnt) < 0) {
                        warnx("mount error: %s", libnvc.error(nvc));
                        goto fail;
                }
                for (size_t i = 0; i < mig_config_devices.ngpus; ++i) {
                        if (libnvc.device_mig_caps_mount(nvc, cnt, mig_config_devices.gpus[i]) < 0) {
                                warnx("mount error: %s", libnvc.error(nvc));
                                goto fail;
                        }
                }
        }
        if (mig_monitor_devices.all && mig_monitor_devices.ngpus) {
                if (libnvc.mig_monitor_global_caps_mount(nvc, cnt) < 0) {
                        warnx("mount error: %s", libnvc.error(nvc));
                        goto fail;
                }
                for (size_t i = 0; i < mig_monitor_devices.ngpus; ++i) {
                        if (libnvc.device_mig_caps_mount(nvc, cnt, mig_monitor_devices.gpus[i]) < 0) {
                                warnx("mount error: %s", libnvc.error(nvc));
                                goto fail;
                        }
                }
        }
        for (size_t i = 0; i < nvc_cfg->imex.nchans; ++i) {
                if (libnvc.imex_channel_mount(nvc, cnt, &nvc_cfg->imex.chans[i]) < 0) {
                        warnx("mount error: %s", libnvc.error(nvc));
                        goto fail;
                }
        }

        /* Update the container ldcache. */
        if (perm_set_capabilities(&err, CAP_EFFECTIVE, ecaps[NVC_LDCACHE], ecaps_size(NVC_LDCACHE)) < 0) {
                warnx("permission error: %s", err.msg);
                goto fail;
        }
        if (libnvc.ldcache_update(nvc, cnt) < 0) {
                warnx("ldcache error: %s", libnvc.error(nvc));
                goto fail;
        }
        /* 7。 更新动态链接缓存
         *  更新容器的动态链接库缓存，使得容器能够正确加载GPU驱动相关的共享库
         */
        if (perm_set_capabilities(&err, CAP_EFFECTIVE, ecaps[NVC_SHUTDOWN], ecaps_size(NVC_SHUTDOWN)) < 0) {
                warnx("permission error: %s", err.msg);
                goto fail;
        }
        rv = EXIT_SUCCESS;

 fail:
        free(nvc_cfg->imex.chans);
        free_devices(&devices);
        libnvc.shutdown(nvc);
        libnvc.container_free(cnt);
        libnvc.device_info_free(dev);
        libnvc.driver_info_free(drv);
        libnvc.container_config_free(cnt_cfg);
        libnvc.config_free(nvc_cfg);
        libnvc.context_free(nvc);
        error_reset(&err);
        return (rv);
}

```
代码原理
- 权限管理：
基于 Linux 的能力（capabilities）模型，控制运行时各个阶段的权限，减少进程的权限范围以提高安全性。
- 设备抽象：
使用 nvc_context 和相关配置对象抽象 GPU 设备和驱动，隐藏底层实现细节。
- 需求验证：
使用规则引擎（DSL）评估容器的需求是否被满足，以确保 GPU 环境与容器配置兼容。
- 资源隔离：
通过挂载特定的设备和功能，将主机的 GPU 资源安全地分配到容器中，支持多容器并发使用 GPU。

libnvidia-container是采用 linux c mount --bind功能将 CUDA Driver Libraries/Binaries 一个个挂载到容器里，而不是将整个目录挂载到容器中。