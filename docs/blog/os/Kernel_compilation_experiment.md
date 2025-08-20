---
title: TLinux内核编译实验
---


## 熟悉内核编译

了解menuconfig，kconfig等。了解内核代码树结构。

### 观察内核代码结构

  ```shell
    tree -L 1
  ```

### 编译内核

  Git branch: lts/5.4.241-1-tlinux4-0017

  1. `make ARCH=x86 tencentconfig`
  2. `make menuconfig`    将模块签名去掉


    ![](../../images/os/Kernel_compilation_experiment/make_menu_config_0.png){.half-width}
    ![](../../images/os/Kernel_compilation_experiment/make_menu_config_1.png){.half-width}

  3. `make -j8`

    > 编译可能会碰到错误，根据提示安装下列包：
    >
    > yum install openssl-devel
    >
    > yum install elfutils-libelf-devel
    >
    > yum install dwarves

  4. `make modules_install`

  5. `make install`
  
    ![](../../images/os/Kernel_compilation_experiment/kernel_version_1.png){.half-width}

  6. `reboot`

    ![](../../images/os/Kernel_compilation_experiment/kernel_version_2.png)


## 编写简单的内核模块

  支持三种模式：<span class="blue">**编译进内核**</span>、<span class="blue">**m**</span>、<span class="blue">**独立编译**</span>


!!! info "编译进内核，m和独立编译"
    <span class="blue">m</span>指在make menuconfig菜单中加载新模块[]中的标识。其中[ ]代表不编译该模块，[*]代表将其加入内核代码树一起编译，[M]为单独编译该模块不加入内核代码树。<span class="blue">m</span>对应最后一个，<span class="blue">编译进内核</span>对应第二个。

    区别 ：编译进内核后用modprobe而不是需要insmod，后两者需要。后两者区别：m在tkernel4文件夹内部，独立编译任意指定文件夹。独立编译和前两者区别：前两者需要配置/kernel/Makefile和 init/Kconfig

    这里只展示<span class="blue">编译进内核</span>和<span class="blue">独立编译</span>，<span class="blue">m</span>大同小异略去

### 编译进内核

  1. 添加模块代码（以添加rue模块为例）

    ```
    kernel/rue/Kconfig
    kernel/rue/Makefile
    kernel/rue/cpu_qos.c
    kernel/rue/io_qos.c
    kernel/rue/mem_qos.c
    kernel/rue/net_qos.c
    kernel/rue/rue_main.c
    ```

  2. 修改 /kernel/Makefile（使make的时候可以编译）

    ```
    @@ -43,6 +43,7 @@ obj-y += irq/
    obj-y += rcu/
    obj-y += livepatch/
    obj-y += dma/
    +obj-y += rue/
    
    obj-$(CONFIG_CHECKPOINT_RESTORE) += kcmp.o
    obj-$(CONFIG_FREEZER) += freezer.o
    ```

  3. 修改 /init/Kconfig（使其加入menuconfig）

    ```
    @@ -373,6 +373,7 @@ config AUDITSYSCALL

    source "kernel/irq/Kconfig"
    source "kernel/time/Kconfig"
    +source "kernel/rue/Kconfig"
    source "kernel/Kconfig.preempt"
    
    menu "CPU/Task time and stats accounting"
    ```
  
  4. Make menuconfig 生成.config文件（如果有模块签名去掉模块签名）
    
    ![](../../images/os/Kernel_compilation_experiment/2141.png){.half-width}
    ![](../../images/os/Kernel_compilation_experiment/2142.png){.half-width}
    ![](../../images/os/Kernel_compilation_experiment/2143.png){.half-width}

  5. 删掉grub的这一行（防止固化）

    ```shell
    [root@TENCENT64 tkernel4]# vim /boot/grub2/grubenv #删掉saved_entry=这一行
    #GRUB Environment Block
    saved_entry=0
    kernelopts=root=UUID=5d6283fe-cde6-459f-810c-88c92336a6d5 ro quiet elevator=noopp
    console=ttyS0,115200 console=tty0 vconsole.keymap=us crashkernel=1800M-64G:256MM
    ,64G-128G:512M,128G-:768M vconsole.font=latarcyrheb-sun16 i8042.noaux net.ifnamee
    s=0 biosdevname=0 intel_idle.max_cstate=1 intel_pstate=disable iommu=pt amd_iommm
    u=on
    boot_success=0
    ```

  6.  `make -j8` `make modules_install` `make install`

    ```shell
    [root@TENCENT64 tkernel4]# make -j8
    scripts/kconfig/conf  --syncconfig Kconfig
      CALL    scripts/checksyscalls.sh
      CALL    scripts/atomic/check-atomics.sh
      DESCEND  objtool
      DESCEND  bpf/resolve_btfids
      CHK     include/generated/compile.h
      AR      kernel/rue/built-in.a
      CC [M]  kernel/rue/rue_main.o
      CC [M]  kernel/rue/cpu_qos.o
      CC [M]  kernel/rue/io_qos.o
      CC [M]  kernel/rue/net_qos.o
      CC [M]  kernel/rue/mem_qos.o
      LD [M]  kernel/rue/rue.o
      UPD     kernel/config_data
      GZIP    kernel/config_data.gz
      ...
      LD      arch/x86/boot/setup.elf
      OBJCOPY arch/x86/boot/setup.bin
      OBJCOPY arch/x86/boot/vmlinux.bin
      BUILD   arch/x86/boot/bzImage
    Setup is 17532 bytes (padded to 17920 bytes).
    System is 9513 kB
    CRC b75ecb2b
    Kernel: arch/x86/boot/bzImage is ready  (#4)
      Building modules, stage 2.
      MODPOST 998 modules
      CC [M]  kernel/rue/rue.mod.o
      LD [M]  kernel/rue/rue.ko

    ```

    我们可以看到[M]就是我们新加入并编译的rue模块

    ```shell
    [root@TENCENT64 tkernel4]# make modules_install
    INSTALL arch/x86/crypto/aesni-intel.ko
    INSTALL arch/x86/crypto/blowfish-x86_64.ko
    ...
    [root@TENCENT64 tkernel4]# modprobe rue
    [root@TENCENT64 tkernel4]# make install
    sh ./arch/x86/boot/install.sh 5.4.241-1+ arch/x86/boot/bzImage \
            System.map "/boot"

    ```

  7. `reboot`

    ```shell
    [root@TENCENT64 tkernel4]# reboot
    ...
    .reboot and login
    ...
    ```

  8. 用`modprobe` `<$module_name>`，`lsmod`和`dmesg`验证模块加载成功

??? success "控制台"
    ```shell
    [root@TENCENT64 ~]# modprobe rue              #无报错表明成功
    [root@TENCENT64 ~]# lsmod
    Module                  Size  Used by
    rue                    16384  0
    sunrpc                397312  1
    virtio_balloon         24576  0
    sch_fq_codel           20480  5
    binfmt_misc            24576  1
    ip_tables              28672  0
    virtio_net             57344  0
    net_failover           20480  1 virtio_net
    failover               16384  1 net_failover
    iscsi_tcp              24576  0
    libiscsi_tcp           32768  1 iscsi_tcp
    libiscsi               61440  2 libiscsi_tcp,iscsi_tcp
    fuse                  118784  1
    autofs4                45056  2

    [root@TENCENT64 ~]# dmesg
    [    0.000000] Linux version 5.4.241-1+ (root@TENCENT64.site) (gcc version 8.4.1 20200928 (Red Hat 8.4.1-1) (GCC)) #4 SMP Thu Aug 3 02:38:31 CST 2023
    [    0.000000] Command line: BOOT_IMAGE=(hd0,msdos1)/boot/vmlinuz-5.4.241-1+ root=UUID=5d6283fe-cde6-459f-810c-88c92336a6d5 ro quiet elevator=noop console=ttyS0,115200 console=tty0 vconsole.keymap=us crashkernel=1800M-64G:256M,64G-128G:512M,128G-:768M vconsole.font=latarcyrheb-sun16 i8042.noaux net.ifnames=0 biosdevname=0 intel_idle.max_cstate=1 intel_pstate=disable iommu=pt amd_iommu=on
    [    0.000000] x86/fpu: x87 FPU will use FXSAVE
    ...
    [    6.493898] startagent.sh (770): /proc/842/oom_adj is deprecated, please use /proc/842/oom_score_adj instead.
    [  163.486416] RUE mod init
    [  163.486417] exp for mod load test on 23.8.2.

    [root@TENCENT64 ~]# rmmod rue
    [root@TENCENT64 ~]# lsmod
    Module                  Size  Used by
    sunrpc                397312  1
    virtio_balloon         24576  0
    sch_fq_codel           20480  5
    binfmt_misc            24576  1
    ip_tables              28672  0
    virtio_net             57344  0
    net_failover           20480  1 virtio_net
    failover               16384  1 net_failover
    iscsi_tcp              24576  0
    libiscsi_tcp           32768  1 iscsi_tcp
    libiscsi               61440  2 libiscsi_tcp,iscsi_tcp
    fuse                  118784  1
    autofs4                45056  2
    [root@TENCENT64 ~]# dmesg
    ...
    [    6.493898] startagent.sh (770): /proc/842/oom_adj is deprecated, please use /proc/842/oom_score_adj instead.
    [  163.486416] RUE mod init
    [  163.486417] exp for mod load test on 23.8.2.
    [  261.490733] RUE mod exit
    ```

### 独立编译

  1. 写模块代码

    ```
    kernel/rue/cpu_qos.c
    kernel/rue/io_qos.c
    kernel/rue/mem_qos.c
    kernel/rue/net_qos.c
    kernel/rue/rue_main.c
    ```

  2. 增加Makefile文件(注意makefile必须用tab不能用空格，不然会报错)

    ```makefile

      #SPDX-License-Identifier: GPL-2.0

      ifneq ($(KERNELRELEASE),)
      #inside kernel tree
      obj-m   := rue.o
      rue-y   := rue_main.o cpu_qos.o io_qos.o net_qos.o mem_qos.o

      else

      KERNEL_DIR ?= /lib/modules/`uname -r`/build

      default:
              $(MAKE) -C $(KERNEL_DIR) M=$$PWD

      clean:
              rm -rf *.o *.o.p *~ core .depend .*.cmd *.ko *.mod.c \
              .tmp_versions Module.symvers modules.order \
              *.a *.mod *.builtin

      endif

    ```

  3. `make` `insmod <$module_name>.ko`

??? success "控制台"
    ```shell
    [root@TENCENT64 rue]# pwd
    /data1/test/rue
    [root@TENCENT64 rue]# lsmod
    Module                  Size  Used by
    sunrpc                397312  1
    virtio_balloon         24576  0
    sch_fq_codel           20480  5
    binfmt_misc            24576  1
    ip_tables              28672  0
    virtio_net             57344  0
    net_failover           20480  1 virtio_net
    failover               16384  1 net_failover
    iscsi_tcp              24576  0
    libiscsi_tcp           32768  1 iscsi_tcp
    libiscsi               61440  2 libiscsi_tcp,iscsi_tcp
    fuse                  118784  1
    autofs4                45056  2
    [root@TENCENT64 rue]# make
    make -C /lib/modules/`uname -r`/build M=$PWD
    make[1]: Entering directory '/data1/tkernel4'
      AR      /data1/test/rue/built-in.a
      CC [M]  /data1/test/rue/rue_main.o
      CC [M]  /data1/test/rue/cpu_qos.o
      CC [M]  /data1/test/rue/io_qos.o
      CC [M]  /data1/test/rue/net_qos.o
      CC [M]  /data1/test/rue/mem_qos.o
      LD [M]  /data1/test/rue/rue.o
      Building modules, stage 2.
      MODPOST 1 modules
      CC [M]  /data1/test/rue/rue.mod.o
      LD [M]  /data1/test/rue/rue.ko
    make[1]: Leaving directory '/data1/tkernel4'
    [root@TENCENT64 rue]# insmod rue.ko
    [root@TENCENT64 rue]# lsmod
    Module                  Size  Used by
    rue                    16384  0
    sunrpc                397312  1
    virtio_balloon         24576  0
    sch_fq_codel           20480  5
    binfmt_misc            24576  1
    ip_tables              28672  0
    virtio_net             57344  0
    net_failover           20480  1 virtio_net
    failover               16384  1 net_failover
    iscsi_tcp              24576  0
    libiscsi_tcp           32768  1 iscsi_tcp
    libiscsi               61440  2 libiscsi_tcp,iscsi_tcp
    fuse                  118784  1
    autofs4                45056  2
    [root@TENCENT64 rue]# dmesg
    [    0.000000] Linux version 5.4.241-1+ (root@TENCENT64.site) (gcc version 8.4.1 20200928 (Red Hat 8.4.1-1) (GCC)) #4 SMP Thu Aug 3 02:38:31 CST 2023
    [    0.000000] Command line: BOOT_IMAGE=(hd0,msdos1)/boot/vmlinuz-5.4.241-1+ root=UUID=5d6283fe-cde6-459f-810c-88c92336a6d5 ro quiet elevator=noop console=ttyS0,115200 console=tty0 vconsole.keymap=us crashkernel=1800M-64G:256M,64G-128G:512M,128G-:768M vconsole.font=latarcyrheb-sun16 i8042.noaux net.ifnames=0 biosdevname=0 intel_idle.max_cstate=1 intel_pstate=disable iommu=pt amd_iommu=on
    [    0.000000] x86/fpu: x87 FPU will use FXSAVE
    ...
    [    7.572456] startagent.sh (773): /proc/909/oom_adj is deprecated, please use /proc/909/oom_score_adj instead.
    [44504.119760] rue: loading out-of-tree module taints kernel.
    [44504.121587] RUE mod init
    [44504.121587] exp for mod load test on 23.8.2.
    [root@TENCENT64 rue]# rmmod rue
    [root@TENCENT64 rue]# lsmod
    Module                  Size  Used by
    sunrpc                397312  1
    virtio_balloon         24576  0
    sch_fq_codel           20480  5
    binfmt_misc            24576  1
    ip_tables              28672  0
    virtio_net             57344  0
    net_failover           20480  1 virtio_net
    failover               16384  1 net_failover
    iscsi_tcp              24576  0
    libiscsi_tcp           32768  1 iscsi_tcp
    libiscsi               61440  2 libiscsi_tcp,iscsi_tcp
    fuse                  118784  1
    autofs4                45056  2
    ```



## 给内核打补丁

  制作自己的测试内核并替换

!!! info "主要体现为对Quilt工具的使用"




???+ success "控制台"
    ```shell
    [root@TENCENT64 tkernel4]# quilt new exp-gy-init-rue-module.patch
    Patch patches/exp-gy-init-rue-module.patch is now on top
    [root@TENCENT64 tkernel4]# quilt series
    patches/exp-gy-init-rue-module.patch
    [root@TENCENT64 tkernel4]# quilt add /kernel/Makefile init/Kconfig
    File /kernel/Makefile added to patch patches/exp-gy-init-rue-module.patch
    [root@TENCENT64 tkernel4]# cd kernel/rue 
    [root@TENCENT64 rue]# touch rue_main.c cpu_qos.c ... #此处省略add其他文件
    [root@TENCENT64 rue]# quilt add rue_main.c cpu_qos.c ... #此处省略add其他文件
    File kernel/rue/rue_main.c added to patch ../../patches/exp-gy-rue-module-init.patch
    [root@TENCENT64 rue]# quilt files
    init/Kconfig
    kernel/Makefile
    kernel/rue/Kconfig
    kernel/rue/Makefile
    kernel/rue/cpu_qos.c
    kernel/rue/io_qos.c
    kernel/rue/mem_qos.c
    kernel/rue/net_qos.c
    kernel/rue/rue_main.c
    [root@TENCENT64 rue]# vi rue_main.c
    ......
    ...修改
    ......
    [root@TENCENT64 tkernel4]# quilt refresh
    Refreshed patch patches/exp-gy-rue-module-init.patch
    [root@TENCENT64 tkernel4]# cat patches/exp-gy-rue-module-init.patch
    Index: tkernel4/kernel/rue/io_qos.c
    ===================================================================
    --- /dev/null
    +++ tkernel4/kernel/rue/io_qos.c
    @@ -0,0 +1,7 @@
    +// SPDX-License-Identifier: GPL-2.0+
    +/*
    + * Tencent RUE IO QoS
    + *
    + * Copyright (c) 2023 Tencent Corporation.
    + *
    + */#
    Index: tkernel4/kernel/rue/rue_main.c
    ===================================================================
    --- /dev/null
    +++ tkernel4/kernel/rue/rue_main.c
    @@ -0,0 +1,26 @@
    +// SPDX-License-Identifier: GPL-2.0+
    +/*
    + * Tencent RUE
    + *
    + * Copyright (c) 2023 Tencent Corporation.
    + *
    + */
    +
    +#include <linux/kernel.h>
    +#include <linux/module.h>
    +
    +static int rue_mod_init(void)
    +{
    +        pr_info("RUE mod init\n");
    +        return 0;
    +}
    +
    +static void rue_mod_exit(void)
    +{
    +        pr_info("RUE mod exit\n");
    +}
    +
    +module_init(rue_mod_init);
    +module_exit(rue_mod_exit);
    +MODULE_AUTHOR("Tencent Corporation");
    +MODULE_LICENSE("GPL v2");
    Index: tkernel4/kernel/rue/Kconfig
    ===================================================================
    --- /dev/null
    +++ tkernel4/kernel/rue/Kconfig
    @@ -0,0 +1,53 @@
    +# Kconfig
    +
    +# SPDX-License-Identifier: GPL-2.0
    +#
    +# Tencent RUE configuration
    +#
    +
    +config RUE
    +    tristate "Resource Utilization Enhancement"
    +    help
    +      Rue provides cgroup level resource isolation. It can help user to
    +      increase resource utilization. Rue contains four subsystems:
    +      cpu, memory, io, network. It allocates resource according to the
    +      priority of the cgroup.
    +
    +config RUE_CPU_QOS
    +    bool "RUE CPU QOS"
    +    depends on RUE
    +    help
    +    This option enables RUE cpu qos function. This adds a bt offline sched
    +    class.
    +
    +    The user can set a croup to offline, then the task in offline cgroup
    +    will only use cpu when the cpu has no online task to run.
    +
    +config RUE_MEM_QOS
    +    bool "RUE MEM QOS"
    +    depends on RUE
    +    help
    +    This option enables RUE memory qos function.
    +
    +    RUE memory qos provides several features. These featues guarantee the
    +    memory allocation of online cgroup. I.E. when the kernel oom, rue
    +    will first kill the cgroup whose priority is the lowest.
    +
    +config RUE_IO_QOS
    +    bool "RUE IO QOS"
    +    depends on RUE
    +    help
    +    This option enables RUE io qos function.
    +
    +    RUE io qos provides several features. These featues guarantee the
    +    io req of online cgroup. And provides the user a read + write bps/iops
    +    throttle feature.
    +
    +config RUE_NET_QOS
    +    bool "RUE NET QOS"
    +    depends on RUE
    +    help
    +    This option enables RUE net qos function.
    +
    +    RUE net qos provides several features. These featues guarantee the
    +    network bandwidth allocation of online cgroup.
    Index: tkernel4/kernel/rue/Makefile
    ===================================================================
    --- /dev/null
    +++ tkernel4/kernel/rue/Makefile
    @@ -0,0 +1,22 @@
    +# SPDX-License-Identifier: GPL-2.0
    +
    +ifneq ($(KERNELRELEASE),)
    +#inside kernel tree
    +obj-m   := rue.o
    +rue-y   := rue_main.o cpu_qos.o io_qos.o net_qos.o mem_qos.o
    +
    +else
    +
    +KERNEL_DIR ?= /lib/modules/`uname -r`/build
    +
    +default:
    +       $(MAKE) -C $(KERNEL_DIR) M=$$PWD
    +
    +clean:
    +       rm -rf *.o *.o.p *~ core .depend .*.cmd *.ko *.mod.c \
    +        .tmp_versions Module.symvers modules.order \
    +        *.a *.mod *.builtin
    +
    +endif
    +
    +
    Index: tkernel4/kernel/rue/cpu_qos.c
    ===================================================================
    --- /dev/null
    +++ tkernel4/kernel/rue/cpu_qos.c
    @@ -0,0 +1,7 @@
    +// SPDX-License-Identifier: GPL-2.0+
    +/*
    + * Tencent RUE CPU QoS
    + *
    + * Copyright (c) 2023 Tencent Corporation.
    + *
    + */
    Index: tkernel4/kernel/rue/mem_qos.c
    ===================================================================
    --- /dev/null
    +++ tkernel4/kernel/rue/mem_qos.c
    @@ -0,0 +1,7 @@
    +// SPDX-License-Identifier: GPL-2.0+
    +/*
    + * Tencent RUE Memory QoS
    + *
    + * Copyright (c) 2023 Tencent Corporation.
    + *
    + */
    Index: tkernel4/kernel/rue/net_qos.c
    ===================================================================
    --- /dev/null
    +++ tkernel4/kernel/rue/net_qos.c
    @@ -0,0 +1,7 @@
    +// SPDX-License-Identifier: GPL-2.0+
    +/*
    + * Tencent RUE NetWork QoS
    + *
    + * Copyright (c) 2023 Tencent Corporation.
    + *
    + */
    Index: tkernel4/init/Kconfig
    ===================================================================
    --- tkernel4.orig/init/Kconfig
    +++ tkernel4/init/Kconfig
    @@ -387,6 +387,7 @@ config AUDITSYSCALL
    
    source "kernel/irq/Kconfig"
    source "kernel/time/Kconfig"
    +source "kernel/rue/Kconfig"
    source "kernel/Kconfig.preempt"
    
    menu "CPU/Task time and stats accounting"
    Index: tkernel4/kernel/Makefile
    ===================================================================
    --- tkernel4.orig/kernel/Makefile
    +++ tkernel4/kernel/Makefile
    @@ -47,6 +47,7 @@ obj-y += irq/
    obj-y += rcu/
    obj-y += livepatch/
    obj-y += dma/
    +obj-y += rue/
    
    obj-$(CONFIG_CHECKPOINT_RESTORE) += kcmp.o
    obj-$(CONFIG_FREEZER) += freezer.o
    
    [root@TENCENT64 rue]# ll
    total 28
    -rw-r--r-- 1 root root  116 Aug  2 21:19 cpu_qos.c
    -rw-r--r-- 1 root root  116 Aug  2 21:19 io_qos.c
    -rw-r--r-- 1 root root 1528 Aug  2 21:19 Kconfig
    -rw-r--r-- 1 root root  409 Aug  2 21:19 Makefile
    -rw-r--r-- 1 root root  119 Aug  2 21:19 mem_qos.c
    -rw-r--r-- 1 root root  120 Aug  2 21:19 net_qos.c
    -rw-r--r-- 1 root root  439 Aug  2 21:19 rue_main.c
    [root@TENCENT64 rue]# quilt pop
    Removing patch ../../patches/exp-gy-rue-module-init.patch
    Restoring kernel/Makefile
    Removing kernel/rue/io_qos.c
    Removing kernel/rue/Makefile
    Removing kernel/rue/cpu_qos.c
    Removing kernel/rue/Kconfig
    Removing kernel/rue/net_qos.c
    Removing kernel/rue/rue_main.c
    Removing kernel/rue/mem_qos.c
    Restoring init/Kconfig

    No patches applied
    [root@TENCENT64 rue]# ll
    total 0
    [root@TENCENT64 rue]# quilt push
    Applying patch ../../patches/exp-gy-rue-module-init.patch
    patching file kernel/rue/io_qos.c
    patching file kernel/rue/rue_main.c
    patching file kernel/rue/Kconfig
    patching file kernel/rue/Makefile
    patching file kernel/rue/cpu_qos.c
    patching file kernel/rue/mem_qos.c
    patching file kernel/rue/net_qos.c
    patching file init/Kconfig
    patching file kernel/Makefile

    Now at patch ../../patches/exp-gy-rue-module-init.patch
    [root@TENCENT64 rue]# ll
    total 28
    -rw-r--r-- 1 root root  116 Aug  2 21:20 cpu_qos.c
    -rw-r--r-- 1 root root  116 Aug  2 21:20 io_qos.c
    -rw-r--r-- 1 root root 1528 Aug  2 21:20 Kconfig
    -rw-r--r-- 1 root root  409 Aug  2 21:20 Makefile
    -rw-r--r-- 1 root root  119 Aug  2 21:20 mem_qos.c
    -rw-r--r-- 1 root root  120 Aug  2 21:20 net_qos.c
    -rw-r--r-- 1 root root  439 Aug  2 21:20 rue_main.c
    ```

