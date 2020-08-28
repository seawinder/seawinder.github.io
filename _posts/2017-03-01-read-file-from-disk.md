---
layout: post
keywords: os, filesystem, 文件系统, 硬盘, 打开文件, 过程
description:  进程从硬盘读取文件时发生了什么
title: "进程从硬盘读取文件的过程"
categories: [OS]
tags: [OS, filesystem]
group: archive
icon: file-alt
---
{% include codepiano/setup %}
在知乎回答的一个问题，贴在这里 [CPU与硬盘关系的几点疑问？ - 中央处理器 (CPU) - 知乎 ](https://www.zhihu.com/question/55954313/answer/148837809)

CPU 和硬盘的关系是不太好描述，CPU 本质上只是用来执行指令，具体的读取文件的操作是操作系统来做的，从操作系统的角度来说可能要方便一些。像其他答案说的，你的这些疑问应该去看操作系统和计算机组成原理相关的教材，形成一个整体上认识，而不应该片面的了解某一个方面。

我下面简单叙述一下操作系统在从硬盘读文件的流程。

为简单起见，假设场景是一个x86体系的32位Linux操作系统中运行的进程 P 需要读取文件 /home/user/test.txt，文件系统使用 ext3。

1. Linux 系统提供了和 IO 相关的系统调用，进程 P 如果要读取文件，首先需要发起系统调用(System Call)  open，传入文件路径"/home/user/test.txt" 和相关参数，来打开文件，执行系统调用以后操作系统会从用户态转换到内核态，部分 CPU 提供了"trap"或者"syscall" 指令来完成状态切换，切换到内核态以后，操作系统调用相应的处理器(handler) 开始处理读取文件的请求。
1. 系统调用 open 并不会直接读取文件内容返回给进程，而是先进行权限方面的检查，如果进程可以访问这个文件，就根据文件路径去查找文件对应的  inode 编号。这部分属于文件系统的内容，在这里简单说一下，每一个非软链接的文件或者目录都具有一个惟一的 inode 编号，对应 inode table 中的一个 inode 数据结构，inode 中包含文件大小，修改时间之类的元信息，也包括一个树形的结构，这个树形结构里面索引了文件内容存储在硬盘的哪些扇区(sector)里。每个目录也有一个对应的磁盘文件来存储该目录中直接包含的子文件或子目录的名字和 inode 编号。比如查找"/home/user/test.txt"，需要按照路径逐级查找，首先根目录 / 的 inode 编号是约定的，为2，操作系统通过 inode 编号2这个信息去 inode table 中找到根目录对应的 inode 信息，根据 inode 信息读取磁盘扇区获取文件内容(这里已经需要访问磁盘了，下面再详细说访问过程)，里面存储的内容比较复杂，为了提升检索效率可能会使用 B+树之类的进行了索引，这里为描述方便，简化一下，比如 根目录下有3个目录home、etc、bin，1个文件 eg.txt，那么根目录对应的文件内容大概类似于
    <pre>
    home 3
    etc 4
    bin 5
    eg.txt 5
    </pre>
后面的数字就是目录或者文件对应的 inode 编号，通过检索可知，我们需要的找到 home 目录的inode 编号为3，重复这个过程，直到定位到 /home/user/test.txt 的 inode 编号。文件系统所有的 inode 信息也是存储在硬盘上的，也就是说硬盘中不仅有文件内容，还有文件系统的数据，这可以解释你的第二、三个问题，为什么换了硬盘仍然能够开机和识别文件，开机是存储在主板上的 BIOS 引导的，操作系统启动以后从硬盘读取文件系统的数据就可以获得整个磁盘的文件信息，CPU 只是执行操作系统的指令。
1. 在获取 inode 以后，操作系统生成了一个文件描述符(file descriptor)，存储在进程 P 自己的 file descriptors 数据结构中，通过文件描述符可以索引到文件的打开方式（只读、读写等）还有 要打开文件的 inode。然后操作系统将文件描述符返回给进程 P，至此系统调用 open 完成。
1. 进程 P 获取到文件 /home/user/test.txt 的描述符以后，还需要再发起系统调用 read，传入文件描述符来读取文件内容，同样读取操作需要切换到内核态由内核代为完成。切换到内核态以后，操作系统通过文件描述符找到对应的 inode ，通过 inode 来确定文件存储在磁盘哪些扇区中，然后向磁盘发送指令来读取这些扇区，把内容读取到内核的地址空间里面。一般来说操作系统会通过 memory mapped IO 技术把键盘、磁盘等硬件上的寄存器连接到 IO 总线，再通过 IO 控制器连接到内存总线，这样硬件上的寄存器也被映射到了一段内存地址上，CPU 可以直接通过读写内存的指令来读写硬件寄存器中的数据。同时还会通过 DMA 技术来让硬件不通过 CPU，直接读写内存的内容，这样磁盘在传输文件的同时 CPU 可以去执行其他线程。
1. 磁盘的 IO 操作完成后，磁盘会触发一个中断(interrupt)，CPU 会暂时中止当前线程的执行，保存相关的寄存器信息后，调用对应的中断处理器(interrupt handler)，把读取到的内容从内核地址空间拷贝到进程 P 的地址空间里面，然后将进程 P的状态设置为 runnable, 进程 P 排队等待自己的 CPU 时间片，被调度器调度以后可以继续执行。