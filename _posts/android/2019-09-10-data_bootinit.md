---
layout: post
title:  Android 开机启动流程至Launcher init篇(一)
category: Android
tags: 开机启动
---
* content
{:toc}

##### 前言
对于Android的开机流程，不管是做为App开发者还是Framework开发者都应该了解的，做为App开发者来说  
了解关于系统的开机流程, 对App的开发也是大有裨益的,对于bug的解决定位更快捷点. 所以我们一起来学习  
下系统的开机流程.

##### 1. 下面是整个流程图
![avatar](https://github.com/skypx/BlogResource/raw/master/other/boot.jpg)


#####  具体的细节流程

###### 1.1 kernel是怎么初始化init进程的,
![avatar](https://github.com/skypx/BlogResource/raw/master/other/initboot.jpg)
可以看到是通过do_execve方法来开启了一个新的进程
源码部分:: <http://androidxref.com/kernel_3.18/xref/init/main.c#run_init_process>

关于内核靠下部分，内核部分初始化了好多驱动的操作以及一些cpu的配置操作，我也是看了个大概就不拿出来现丑了.

查看kernel log的方法，启动后进入adb模式,然后命令:: **dmesg**


###### 1.2 init 进程
然后我们可以看到从kernel的kernel_init方法里面启动一个进程就是init进程，就是手机根目录下的init可执行程序

一切都始于init，bootloader 加载了内核，内核启动了init 进程。Linux系统中的init进程(pid=1)是除了idle进程(pid=0，也就是init_task)之外另一个比较特殊的进程，它是Linux内核开始建立起进程概念时第一个通过kernel_thread产生的进程，其开始在内核态执行，然后通过一个系统调用，开始执行用户空间的/sbin/init程序，期间Linux内核也经历了从内核态到用户态的特权级转变，然后所有的用户进程都有该进程派生出来。

init进程的源代码位于**system/core/init**
<http://androidxref.com/9.0.0_r3/xref/system/core/init/init.cpp>  
查看mk文件得到编译生成的bin路径: LOCAL_MODULE_PATH := $(TARGET_ROOT_OUT)  


```java
int main(int argc, char** argv) {
546    if (!strcmp(basename(argv[0]), "ueventd")) {
           //创建设备节点,在这里会解析, ueventd.rc
           //ueventd.rc 这个文件里面会有设备节点,类似dev/
547        return ueventd_main(argc, argv);
548    }
549
550    if (!strcmp(basename(argv[0]), "watchdogd")) {
           //watchdogd俗称看门狗，用于系统出问题时重启系统
551        return watchdogd_main(argc, argv);
552    }
553
554    if (argc > 1 && !strcmp(argv[1], "subcontext")) {
555        InitKernelLogging(argv);
556        const BuiltinFunctionMap function_map;
557        return SubcontextMain(argc, argv, &function_map);
558    }
559
560    if (REBOOT_BOOTLOADER_ON_PANIC) {
            //初始化重启系统的处理信号，内部通过sigaction 注册信号，当监听到该信号时重启系统
561        InstallRebootSignalHandlers();
562    }
563
564    bool is_first_stage = (getenv("INIT_SECOND_STAGE") == nullptr);
565     //第一阶段:: 这个时候为true，
        //挂载文件系统并创建目录
566    if (is_first_stage) {
567        boot_clock::time_point start_time = boot_clock::now();
568
569        // Clear the umask.
570        umask(0);
571
572        clearenv();
573        setenv("PATH", _PATH_DEFPATH, 1);
574        // Get the basic filesystem setup we need put together in the initramdisk
575        // on / and then we'll let the rc file figure out the rest.
576        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
577        mkdir("/dev/pts", 0755);
578        mkdir("/dev/socket", 0755);
579        mount("devpts", "/dev/pts", "devpts", 0, NULL);
580        #define MAKE_STR(x) __STRING(x)
581        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
582        // Don't expose the raw commandline to unprivileged processes.
583        chmod("/proc/cmdline", 0440);
584        gid_t groups[] = { AID_READPROC };
585        setgroups(arraysize(groups), groups);
586        mount("sysfs", "/sys", "sysfs", 0, NULL);
587        mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
588
589        mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11));
590
591        if constexpr (WORLD_WRITABLE_KMSG) {
592            mknod("/dev/kmsg_debug", S_IFCHR | 0622, makedev(1, 11));
593        }
594				 //mknod mknod用于创建Linux中的设备文件
					 path：设备所在目录
					 mode：指定设备的类型和读写访问标志
					 可能的类型

					 S_IFMT	type of file ，文件类型掩码
					 S_IFREG	regular 普通文件
					 S_IFBLK	block special 块设备文件
					 S_IFDIR	directory 目录文件
					 S_IFCHR	character special 字符设备文件
					 S_IFIFO	fifo 管道文件
					 S_IFNAM	special named file 特殊文件
					 S_IFLNK	symbolic link 链接文件

595        mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8));
596        mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9));
597
598        // Mount staging areas for devices managed by vold
599        // See storage config details at http://source.android.com/devices/storage/
           //mount属于Linux系统调用
					 mount 命令参数：

					 source：将要挂上的文件系统，通常是一个设备名。
					 target：文件系统所要挂载的目标目录。
					 filesystemtype：文件系统的类型，可以是"ext2"，"msdos"，"proc"，"ntfs"，"iso9660"。

600        mount("tmpfs", "/mnt", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
601              "mode=0755,uid=0,gid=1000");
602        // /mnt/vendor is used to mount vendor-specific partitions that can not be
603        // part of the vendor partition, e.g. because they are mounted read-write.

					 //mkdir也是Linux系统调用，作用是创建目录，第一个参数是目录路径，第二个是读写权限
604        mkdir("/mnt/vendor", 0755);
605
606        // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
607        // talk to the outside world...
608        InitKernelLogging(argv);
609
610        LOG(INFO) << "init first stage started!";
611
612        if (!DoFirstStageMount()) {
613            LOG(FATAL) << "Failed to mount required partitions early ...";
614        }
615
616        SetInitAvbVersionInRecovery();
617
618        // Enable seccomp if global boot option was passed (otherwise it is enabled in zygote).
619        global_seccomp();
620
621        // Set up SELinux, loading the SELinux policy.
622        SelinuxSetupKernelLogging();
623        SelinuxInitialize();
624
625        // We're in the kernel domain, so re-exec init to transition to the init domain now
626        // that the SELinux policy has been loaded.
627        if (selinux_android_restorecon("/init", 0) == -1) {
628            PLOG(FATAL) << "restorecon failed of /init failed";
629        }
630				 //设置第二阶段的变量
631        setenv("INIT_SECOND_STAGE", "true", 1);
632
633        static constexpr uint32_t kNanosecondsPerMillisecond = 1e6;
634        uint64_t start_ms = start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;
635        setenv("INIT_STARTED_AT", std::to_string(start_ms).c_str(), 1);
636
637        char* path = argv[0];
638        char* args[] = { path, nullptr };
					 //重新执行main函数, exec是函数族提供了一个在进程中执行另一个进程的方法
639        execv(path, args);
640
641        // execv() only returns if an error happened, in which case we
642        // panic and never fall through this conditional.
643        PLOG(FATAL) << "execv(\"" << path << "\") failed";
644    }
645
646    // At this point we're in the second stage of init.

       把log写入dev/kmsg中
647    InitKernelLogging(argv);
648    LOG(INFO) << "init second stage started!";
649
650    // Set up a session keyring that all processes will have access to. It
651    // will hold things like FBE encryption keys. No process should override
652    // its session keyring.
653    keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1);
654
655    // Indicate that booting is in progress to background fw loaders, etc.
656    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
657
658    property_init();
659
660    // If arguments are passed both on the command line and in DT,
661    // properties set in DT always have priority over the command-line ones.
662    process_kernel_dt();
663    process_kernel_cmdline();
664
665    // Propagate the kernel variables to internal variables
666    // used by init as well as the current required properties.
667    export_kernel_boot_props();
668
669    // Make the time that init started available for bootstat to log.
670    property_set("ro.boottime.init", getenv("INIT_STARTED_AT"));
671    property_set("ro.boottime.init.selinux", getenv("INIT_SELINUX_TOOK"));
672
673    // Set libavb version for Framework-only OTA match in Treble build.
674    const char* avb_version = getenv("INIT_AVB_VERSION");
675    if (avb_version) property_set("ro.boot.avb_version", avb_version);
676
			 //清除所设置的变量
677    // Clean up our environment.
678    unsetenv("INIT_SECOND_STAGE");
679    unsetenv("INIT_STARTED_AT");
680    unsetenv("INIT_SELINUX_TOOK");
681    unsetenv("INIT_AVB_VERSION");
682
683    // Now set up SELinux for second stage.
684    SelinuxSetupKernelLogging();
685    SelabelInitialize();
686    SelinuxRestoreContext();
687
688    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
689    if (epoll_fd == -1) {
690        PLOG(FATAL) << "epoll_create1 failed";
691    }
692
693    sigchld_handler_init();
694
695    if (!IsRebootCapable()) {
696        // If init does not have the CAP_SYS_BOOT capability, it is running in a container.
697        // In that case, receiving SIGTERM will cause the system to shut down.
698        InstallSigtermHandler();
699    }
700
701    property_load_boot_defaults();
702    export_oem_lock_status();
703    start_property_service();
704    set_usb_controller();
705
706    const BuiltinFunctionMap function_map;
707    Action::set_function_map(&function_map);
708
709    subcontexts = InitializeSubcontexts();
710
711    ActionManager& am = ActionManager::GetInstance();
712    ServiceList& sm = ServiceList::GetInstance();
713
			 //解析init文件,init.rc文件,然后根据文件里面的配置来启动进程native或者service
			 //一般情况下这些service是daemon 也就是守护进程
714    LoadBootScripts(am, sm);
715
716    // Turning this on and letting the INFO logging be discarded adds 0.2s to
717    // Nexus 9 boot time, so it's disabled by default.
718    if (false) DumpState();
719
720    am.QueueEventTrigger("early-init");
721
722    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
723    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
724    // ... so that we can start queuing up actions that require stuff from /dev.
725    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");
726    am.QueueBuiltinAction(SetMmapRndBitsAction, "SetMmapRndBits");
727    am.QueueBuiltinAction(SetKptrRestrictAction, "SetKptrRestrict");
728    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
729    am.QueueBuiltinAction(console_init_action, "console_init");
730
731    // Trigger all the boot actions to get us started.
732    am.QueueEventTrigger("init");
733
734    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
735    // wasn't ready immediately after wait_for_coldboot_done
736    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");
737
738    // Don't mount filesystems or start core system services in charger mode.
739    std::string bootmode = GetProperty("ro.bootmode", "");
740    if (bootmode == "charger") {
741        am.QueueEventTrigger("charger");
742    } else {
743        am.QueueEventTrigger("late-init");
744    }
745
746    // Run all property triggers based on current state of the properties.
747    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");
748
       //进入loop循环，保证init进程不会退出
749    while (true) {
750        // By default, sleep until something happens.
751        int epoll_timeout_ms = -1;
752
753        if (do_shutdown && !shutting_down) {
754            do_shutdown = false;
755            if (HandlePowerctlMessage(shutdown_command)) {
756                shutting_down = true;
757            }
758        }
759
760        if (!(waiting_for_prop || Service::is_exec_service_running())) {
761            am.ExecuteOneCommand();
762        }
763        if (!(waiting_for_prop || Service::is_exec_service_running())) {
764            if (!shutting_down) {
765                auto next_process_restart_time = RestartProcesses();
766
767                // If there's a process that needs restarting, wake up in time for that.
768                if (next_process_restart_time) {
769                    epoll_timeout_ms = std::chrono::ceil<std::chrono::milliseconds>(
770                                           *next_process_restart_time - boot_clock::now())
771                                           .count();
772                    if (epoll_timeout_ms < 0) epoll_timeout_ms = 0;
773                }
774            }
775
776            // If there's more work to do, wake up again immediately.
777            if (am.HasMoreCommands()) epoll_timeout_ms = 0;
778        }
779
780        epoll_event ev;
781        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
782        if (nr == -1) {
783            PLOG(ERROR) << "epoll_wait failed";
784        } else if (nr == 1) {
785            ((void (*)()) ev.data.ptr)();
786        }
787    }
788
789    return 0;
790}
791
792}  // namespace init
793}  // namesp
```
