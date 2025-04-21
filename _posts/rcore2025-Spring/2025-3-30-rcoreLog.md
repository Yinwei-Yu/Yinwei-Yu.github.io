---
layout: post
title: rcore学习日志
subtitle: 从入门到入土
author: 渚汐
date: 2025-3-30
tags:
  - rcore
---

## 3-30

今天训练营开幕,在参加训练营之前自己做到了ch2就没进行下去了,希望可以通过训练营督促自己继续进步.

## 3-31

今天因为有cpu的实验课,时间比较仓促,只简单看了下ch3的描述,还有很多没有搞明白,比如RR的实现,还有switch的实现等,明天时间比较多要仔细看一下.今天复习了一下操作系统课的进程调度相关内容,应该对接下来的学习有所帮助.

## 4-1

今天试图搞清楚ch3的操作系统是如何从头开始一步步执行的,发现对特权级指令和csr的理解还有一些问题,先回第一二章复习一下.

### rust指令

```rust
#![no_std] //不包含标准库
#![no_main] //不包含一般意义的main()函数
#![no_mangle] //编译链接时不修改符号名
#![link_section = "..."] //在链接时的符号
#[linkage = "weak"] //标记为弱链接,若有重名,采用另一个
```

### 汇编相关

U:用户态
S:特权级
M:机器级,RustSBI运行在这里

```asm
ecall //特权级指令,从U->S,S->M
```

CSR寄存器(control and status register)

|CSR名|与Trap相关的功能|
|---|---|
|sstatus|spp字段给出trap前cpu位于哪个特权级|
|sepc|当trap为异常时,记录trap前执行最后一条指令的地址|
|scause|trap原因|
|stval|trap附加信息|
|stvec|trap处理代码的入口地址|

当一个cpu执行完一条指令准备trap到S时:

1. spp字段被修改为相应特权级
2. spec和scause等寄存器存入相关值
3. cpu跳转到stvec指定的地址进行trap处理

在trap时,需要保存sstatus和sepc,因为二者在trap全程有意义(在中断嵌套时还可能被覆盖),而其他的在trap后立刻就被使用了.

时间还是太有限了..关于TaskManager的部分明天再看吧

## 4-2

### time定时器的分析

在主函数中有这么一行:

```rust
os/src/main.rs
timer::set_next_trigger();
```

这个函数拆开后,关键点在这里:

```rust
pub fn set_next_trigger() {
    set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC);
}

pub fn set_timer(timer: usize) {
    sbi_call(SBI_SET_TIMER, timer, 0, 0);
}
```

`get_time()`函数获得自上电以来的cpu周期数,那么传给set_timer的参数值就是当前cpu周期数加上10ms后的cpu周期数,在`set_timer`函数里,调用sbi_call,通过ecall指令进入M状态,进入trap,跳转到trap_handler里获得trap原因参数然后设置下一次trap时间后将当前应用状态设置为就绪.
而发生trap后跳转到的地址是由`stvec`寄存器指定的,这个寄存器的值在初始化时被初始化为trap.S文件中的`__alltraps`,这样完成硬件层级的切换

### TaskManager的分析

在`TASKMANAGER`初始化中难以理解的部分在这儿:

```rust
for (i, task) in tasks.iter_mut().enumerate() {
            task.task_cx = TaskContext::goto_restore(init_app_cx(i));
            task.task_status = TaskStatus::Ready;
        };

/// get app info with entry and sp and save `TrapContext` in kernel stack
pub fn init_app_cx(app_id: usize) -> usize {
    KERNEL_STACK[app_id].push_context(TrapContext::app_init_context(
        get_base_i(app_id),
        USER_STACK[app_id].get_sp(),
    ))
}

pub fn app_init_context(entry: usize, sp: usize) -> Self {
        let mut sstatus = sstatus::read(); // CSR sstatus
        sstatus.set_spp(SPP::User); //previous privilege mode: user mode
        let mut cx = Self {
            x: [0; 32],
            sstatus,
            sepc: entry, // entry point of app
        };
        cx.set_sp(sp); // app's user stack pointer
        cx // return initial Trap Context of app
    }
```

`init_app_cx`函数接受一个app_id,在`内核栈`上分配应用i的trap上下文.分配前先初始化上下文,在`app_init_context`函数中,读取当前模式状态信息,并设置为用户态,然后获得应用的栈指针位置,入口地址设置为应用的起始地址`get_base_i`函数获得.

核心就是通过栈的地址空间,初始化每个应用自己的寄存器,设置csr等,这里涉及到用户栈和内核栈的交互.

## 4-3

没学新内容,做了下作业,明天要清明了,可能会出去丸惹

## 4-7

今天先初看下ch4,争取把代码都搞懂,可能做不了太多,因为今天太多精力要投入到cpu实验里

### address模块遇到的困惑

```rust
impl From<usize> for PhysPageNum {
    fn from(v: usize) -> Self {
        Self(v & ((1 << PPN_WIDTH_SV39) - 1))
    }
}
//PPN_WIDTH_SV39 = 44
```

在riscv sv39标准中,物理页号应该是12到56位的44位,为什么这里是低44位?
因为这里把物理页号作为独立的数值进行处理,不是看做一个地址中的某些位数.

```rust
impl From<VirtAddr> for usize {
    fn from(v: VirtAddr) -> Self {
        if v.0 >= (1 << (VA_WIDTH_SV39 - 1)) {
            v.0 | (!((1 << VA_WIDTH_SV39) - 1))
        } else {
            v.0
        }
    }
}
```

为什么从虚拟地址转换为整数这么麻烦?

因为sv39要求从39-63位要和第38位完全相同.第一个if条件检查第38位是否为1,如果为1,那么创建一个掩码,它的低39位全为0,高位全为1,从而实现了sv39的要求.
如果第38位不是1,直接返回即可.

### page_table模块遇到的困惑

具体的多级页表寻址过程:

从stap寄存器读取到该应用的根页表基址(左移12位之后),然后加上vpn[2]的偏移量获得二级页表的入口物理地址,然后重复上述过程直到找到具体物理地址.

## 4-8

今天读完了ch4的全部内容,尝试做了下实验习题,有点困难..

## 4-9

完成了ch4实验,进度有点慢,要加速了


## 4-10

第五章进程切换相对简单一些,一天差不多六个小时搞完了.

## 4-11

理论知识跟不上了,先花两天补一下文件系统的理论知识吧

## 4-12 4-13

休息

## 4-14

尝试理解ch6,内容比较繁杂,各种封装有点多啊...
没太看懂,明天继续,今天先把文件系统笔记做了再说

## 4-15

做了下文件系统的笔记,对Inode和磁盘分配清晰了不少

## 4-16

杂事缠身,抽不出时间 QAQ

## 4-17

重新看了一遍ch6,感觉还是云里雾里,太复杂了

## 4-18

经过不懈努力终于完成ch6了...
rust Trait 我们喜欢你...

## 4-21

开始ch7,感觉不是很复杂?也没有编程作业,好耶
