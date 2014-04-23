---
layout: post
title: "unwind_frame函数导致内核内存访问错误"
date: 2014-04-23 13:43:06 +0800
comments: true
categories: Linux-kernel
---

## 环境
* 内核版本： 3.0.8
* CPU信息：
```
# cat /proc/cpuinfo 
Processor       : ARMv7 Processor rev 0 (v7l)
processor       : 0
BogoMIPS        : 1849.75

processor       : 1
BogoMIPS        : 1856.30

Features        : swp half thumb fastmult vfp edsp vfpv3 vfpv3d16 
CPU implementer : 0x41
CPU architecture: 7
CPU variant     : 0x3
CPU part        : 0xc09
CPU revision    : 0
```

## 问题
2013年8月份的时候，在一些板子(海思3531平台)上面出现了一些奇怪的问题，查看dmesg都出现了如下打印：
```
[283986.090810] Unable to handle kernel paging request at virtual address fffffff3
[283986.090844] pgd = 848b0000
[283986.090855] [fffffff3] *pgd=8cdfe821, *pte=00000000, *ppte=00000000
[283986.090878] Internal error: Oops: 17 [#1] SMP
[283986.090891] Modules linked in: option usb_wwan usbserial hi_ir 8192cu nvp1918 buzzer led regpro debugs_drv ds3231 0104 msdos vfat fat configfs raid1 md_mod sg sr_mod
m ahci_sys libahci sd_mod libata scsi_wait_scan scsi_mod hi3531_adec(P) hi3531_aenc(P) hi3531_ao(P) hi3531_ai(P) hi3531_sio(P) hidmac jpeg hi3531_hdmi(P) hi3531_vfmw(P) h
_vda(P) hi3531_ive(P) hi3531_region(P) hi3531_vpss(P) hi3531_vou(P) hi3531_viu(P) hi3531_mpeg4e(P) hi3531_jpege(P) hi3531_rc(P) hi3531_h264e(P) hi3531_chnl(P) hi3531_grou
(P) hi3531_tde hi3531_sys hi3531_base(P) mmz pca955_driver i2c_module stmmac
[283986.091116] CPU: 0    Tainted: P             (3.0.8 #1)
[283986.091148] PC is at unwind_frame+0x48/0x68
[283986.091163] LR is at get_wchan+0x8c/0x298
[283986.091178] pc : [<8003d120>]    lr : [<8003a660>]    psr: a0000013
[283986.091183] sp : 824dbcc8  ip : 00000003  fp : 824dbcd4
[283986.091204] r10: 00000001  r9 : 00000000  r8 : 80493c34
[283986.091219] r7 : 00000000  r6 : 00000000  r5 : 8b64df40  r4 : 824dbcd8
[283986.091235] r3 : 824dbcd8  r2 : ffffffff  r1 : 80490000  r0 : 8049003f
[283986.091252] Flags: NzCv  IRQs on  FIQs on  Mode SVC_32  ISA ARM  Segment user
[283986.091271] Control: 10c53c7d  Table: 848b004a  DAC: 00000015
[283986.091285]
----cuted----
[283986.094045] Process pidof (pid: 18711, stack limit = 0x824da2f0)
[283986.09[283986.094663] 
----cuted----
Backtrace:
[283986.094685] [<8003d0d8>] (unwind_frame+0x0/0x68) from [<8003a660>] (get_wchan+0x8c/0x298)
[283986.094714] [<8003a5d4>] (get_wchan+0x0/0x298) from [<8011f700>] (do_task_stat+0x548/0x5ec)
[283986.094733]  r4:00000000
[283986.094750] [<8011f1b8>] (do_task_stat+0x0/0x5ec) from [<8011f7c0>] (proc_tgid_stat+0x1c/0x24)
[283986.094787] [<8011f7a4>] (proc_tgid_stat+0x0/0x24) from [<8011b7f0>] (proc_single_show+0x54/0x98)
[283986.094823] [<8011b79c>] (proc_single_show+0x0/0x98) from [<800e9024>] (seq_read+0x1b4/0x4e4)
[283986.094842]  r8:824dbf08 r7:824dbf70 r6:00000001 r5:8673bbe0 r4:86c88120
[283986.094864] r3:00000000
[283986.094888] [<800e8e70>] (seq_read+0x0/0x4e4) from [<800c8c54>] (vfs_read+0xb4/0x19c)
[283986.094913] [<800c8ba0>] (vfs_read+0x0/0x19c) from [<800c8e18>] (sys_read+0x44/0x74)
[283986.094931]  r8:00000000 r7:00000003 r6:000003ff r5:7ebe3818 r4:8673bbe0
[283986.094963] [<800c8dd4>] (sys_read+0x0/0x74) from [<800393c0>] (ret_fast_syscall+0x0/0x30)
[283986.094982]  r9:824da000 r8:80039568 r6:7ebe3c8f r5:0000000e r4:7ebe3818
[283986.095011] Code: e3c10d7f e3c0103f e151000c 9afffff6 (e512100c)
[283986.095094] ---[ end trace bc07a2a11408a6af ]---
```

## 分析
针对上面的dmesg信息进行分析后，得出了如下结论：

1. 导致出错的当前(current)进程是`pidof`
2. pidof程序的原理就是：遍历所有进程的stat(/proc/PID/stat)文件，找出和参数一样的进程ID(pid)。
3. /proc/PID/stat文件的内容是由`do_task_stat`(见fs/proc/array.c)函数生成
4. `do_task_stat`会调用`get_wchan`(见arch/arm/kernel/process.c)，来获取进程的[Waiting Channnel](http://askubuntu.com/questions/19442/what-is-the-waiting-channel-of-a-process)
5. `get_wchan`函数使用`unwind_frame`(见arch/arm/kernel/stacktrace.c)函数回溯进程的栈指针（fp/sp)，试图获取进程最后一个`非调度函数`的函数调用
6. `unwind_frame`却导致了一个内核paging错误

根据上面的初步分析，结合我们自己的应用程序，推测造成这个问题的应用层场景如下：
1. 有两个进程p1/busy_worker
2. busy_worker有30多个线程
3. p1不停执行system(pidof("busy_worker"))

根据内核打印出来的PC/LR寄存器，可以知道，是unwind_frame函数中出现的内存访问错误：
```
[734212.113523] PC is at unwind_frame+0x48/0x68
```

反汇编内核代码（#号开头的行， 是c语言注释）
```
8003d0d8 <unwind_frame>:
8003d0d8:       e1a0c00d        mov     ip, sp
8003d0dc:       e92dd800        push    {fp, ip, lr, pc}
8003d0e0:       e24cb004        sub     fp, ip, #4

#r1 = frame->sp
#low = frame->sp;
8003d0e4:       e5901004        ldr     r1, [r0, #4]

#r3 = r0 = frame
8003d0e8:       e1a03000        mov     r3, r0

#r2 = unsigned long fp = frame->fp;
8003d0ec:       e5902000        ldr     r2, [r0]

#r0 = low + 12
8003d0f0:       e281000c        add     r0, r1, #12

#if (fp < low+12)
8003d0f4:       e1520000        cmp     r2, r0
8003d0f8:       2a000001        bcs     8003d104 <unwind_frame+0x2c>
8003d0fc:       e3e00015        mvn     r0, #21
8003d100:       e89da800        ldm     sp, {fp, sp, pc}
8003d104:       e2810d7f        add     r0, r1, #8128   ; 0x1fc0
8003d108:       e282c004        add     ip, r2, #4
8003d10c:       e280103f        add     r1, r0, #63     ; 0x3f
8003d110:       e3c10d7f        bic     r0, r1, #8128   ; 0x1fc0
8003d114:       e3c0103f        bic     r1, r0, #63     ; 0x3f
8003d118:       e151000c        cmp     r1, ip
8003d11c:       9afffff6        bls     8003d0fc <unwind_frame+0x24>

#r1 = *(r2-12)
#r1 = *(fp-12)
8003d120:       e512100c        ldr     r1, [r2, #-12]

8003d124:       e3a00000        mov     r0, #0

#frame->fp = *(unsigned long *)(fp - 12);
8003d128:       e5831000        str     r1, [r3]

8003d12c:       e512c008        ldr     ip, [r2, #-8]
8003d130:       e583c004        str     ip, [r3, #4]
8003d134:       e5122004        ldr     r2, [r2, #-4]
8003d138:       e583200c        str     r2, [r3, #12]
8003d13c:       e89da800        ldm     sp, {fp, sp, pc}
```

反汇编结合内核源代码， 可知出错的时候执行到下面的语句：
```c
	/* restore the registers from the stack frame */
	frame->fp = *(unsigned long *)(fp - 12);
```
出错时候的平台寄存器如下：
```
[734212.113557] pc : [<8003d120>]    lr : [<8003a660>]    psr: a0000013
[734212.113561] sp : 845d1cc8  ip : 00000003  fp : 845d1cd4
[734212.113583] r10: 00000001  r9 : 00000000  r8 : 80493c34
[734212.113597] r7 : 00000000  r6 : 00000000  r5 : 83354960  r4 : 845d1cd8
[734212.113613] r3 : 845d1cd8  r2 : ffffffff  r1 : 80490000  r0 : 8049003f
```
在出现的内存访问错误之前， fp寄存器值已经保存在寄存器r2中了， 对应的值是`ffffffff`。 
也就是说， 传递给unwind_frame函数的stackframe->fp = 0xffffffff。
出现问题的情况下应用层的用法都是没有问题的，所以这应该是一个内核BUG。
虽然是很清楚导致此内核Bug的原因，但是相关内核函数都没有处理竞争情况，当时就怀疑这个BUG应该和抢占有关系。
由于不太确定，就去Stack Overflow上面问了个问题：
[unwind_frame cause a kernel paging error](http://stackoverflow.com/questions/18479894/unwind-frame-cause-a-kernel-paging-error)，不过也一直没有任何有价值的回复。

## 巧合
时隔3个月的后，又想起了这个问题，于是去kernel git上搜索了一番，发现2013年后针对unwind_frame函数的修改挺多。
其中[Konstantin Khlebnikov提交的一个patch](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=1b15ec7a7427d4188ba91b9bbac696250a059d22)就解决了这个内核Bug。
更巧的是：修改日志中引用了我在Stack Overflow上提的问题。

## Root cause
Konstantin Khlebnikov在修改日志中写道：
> get_wchan() is lockless. Task may wakeup at any time and change its own stack, thus each next stack frame may be overwritten and filled with random stuff.
/proc/$pid/stack interface had been disabled for non-current tasks, see [1] But 'wchan' still allows to trigger stack frame unwinding on volatile stack.
This patch fixes oops in unwind_frame() by adding stack pointer validation on each step (as x86 code do), unwind_frame() already checks frame pointer.  

大意就是：
get_wchan()是没有加锁的，而进程可以在任何时候执行并修改自己的栈，所以进程的栈可能会被修改并填充随机的值。

假设我们的pidof进程PID为100，记做pidof(100)，busy_worker进程PID为99，记做busy_worker(99), 我们碰到的问题就是：
* pidof(100)通过读/proc/99/stat文件触发了内核去获取busy_worker(99)的Waiting Channnel(get_wchan)
* busy_worker(99)执行到获取文件系统信息时候被抢占，此时其调用栈看起来可能是这个样子：
```
main
    busy_worker_main
        fun1
            fun2
                fun3
                   __kernel_routines_xxxx__
```
* get_wchan将busy_worker(99)的fp/sp/lr/pc都保存在stackframe结构体中
```c
	frame.fp = thread_saved_fp(p);
	frame.sp = thread_saved_sp(p);
	frame.lr = 0;			/* recovered from the stack */
	frame.pc = thread_saved_pc(p);
```
* 由于没有锁，此时busy_worker(99)又开始在CPU上执行（抢占），它可能又去执行其他功能代码，调用栈可能是：
```
main
    busy_worker_main
        fun5
            fun6
                fun7
                   __kernel_routines_yyyy__
```
* 内核调用unwind_frame开始回溯busy_worker_main(99)的栈。
这个时候问题出现了，此时busy_worker_main(99)的栈已经和第2步中完全不一样了，内核再调用下面语句去访问`上次记录`的栈里面内存，有点类似于访问已经释放了的内存：
```c
	frame->fp = *(unsigned long *)(fp - 12);
	frame->sp = *(unsigned long *)(fp - 8);
	frame->pc = *(unsigned long *)(fp - 4);
```
这个时候，这块内存保存的可能是某个函数的局部变量、或者被push进来的其他寄存器。
总而言之，访问这块内存会得到一个随机值，而这里恰好是0xffffffff，导致了内核访问错误的内存，从而出现paging error.

## 参考
* [ARM: 7912/1: check stack pointer in get_wchan](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=1b15ec7a7427d4188ba91b9bbac696250a059d22)
* [What is the “Waiting Channel” of a process?](http://askubuntu.com/questions/19442/what-is-the-waiting-channel-of-a-process)
* [Arm registers](/images/arm_registers.png)

