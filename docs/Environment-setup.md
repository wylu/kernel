# 1. 环境设置

我们需要一个基础来设计和制造我们的内核。在这里，我假设您正在使用带有GNU工具链的 `*nix` 系统。如果要使用Windows系统，则必须使用cygwin（这是 `*nix` 仿真环境）或DJGPP。无论哪种方式，本教程中的makefile和命令都可能不起作用。

## 1.1. 目录结构

我的目录布局如下：

```
tutorial
 |
 +-- src
 |
 +-- docs
```

您所有的源文件都将放在src中，并且您所有的文档（您确实写过文档吗？;）都应该放在docs中。

## 1.2. 编译

本教程中的示例应使用GNU工具链（gcc，ld，gas等）成功编译。汇编示例使用intel语法编写，我个人认为，它比GNU AS使用的AT＆T语法更具可读性。要组装它们，您将需要[Netwide Assembler](https://www.nasm.us/)。

本教程不是bootloader教程。我们将使用[GRUB](http://www.gnu.org/software/grub/)加载内核。为此，我们需要一个预加载了GRUB的软盘映像。有一些教程可以做到这一点，但幸运的是，我已经制作了一个标准img，可以在这里找到。这放在您的'tutorial'（或您命名的任何目录）目录中。

## 1.3. 运行

没有其他选择可以将裸机作为测试平台系统。不幸的是，裸露的硬件很容易告诉您哪里出了问题（但是，当然，您将第一次编写完全没有错误的代码，不是吗？）。进入[Bochs](http://bochs.sourceforge.net/)。Bochs是一个开源x86-64仿真器。当事情完全出错时，bochs会告诉您，并将处理器状态存储在日志文件中，这非常有用。而且它的运行和重新启动速度都比真实计算机快得多。我的示例将在boch上很好地运行。

## 1.4. Bochs

为了运行bochs，您将需要一个bochs配置文件（bochsrc.txt）。巧合的是，下面包含一个示例！

请注意BIOS文件的位置。这些安装之间似乎有所变化，如果您从源头制作boch，很可能甚至没有。Googlet它们文件名，您可以从官方的bochs网站获得它们。

```yaml
megs: 32
romimage: file=/usr/share/bochs/BIOS-bochs-latest, address=0xf0000
vgaromimage: /usr/share/bochs/VGABIOS-elpin-2.40
floppya: 1_44=/dev/loop0, status=inserted
boot: a
log: bochsout.txt
mouse: enabled=0
clock: sync=realtime
cpu: ips=500000
```

这将使bochs仿真32MB机器，其时钟速度类似于350MHz PentiumII。可以提高每秒的指令速度-我更喜欢较慢的仿真速度，只是这样我就可以看到滚动大量文本时发生了什么。

## 1.5. 实用的脚本

我们将经常做几件事-making (compiling and linking)我们的项目，并将生成的内核二进制文件传输到我们的软盘映像中。

### 1.5.1. Makefile

```makefile
# Makefile for JamesM's kernel tutorials.
# The C and C++ rules are already setup by default.
# The only one that needs changing is the assembler 
# rule, as we use nasm instead of GNU as.

SOURCES=boot.o

CFLAGS=
LDFLAGS=-Tlink.ld
ASFLAGS=-felf

all: $(SOURCES) link 

clean:
 » 	-rm *.o kernel

link:
 » 	ld $(LDFLAGS) -o kernel $(SOURCES)

.s.o:
 » 	nasm $(ASFLAGS) $<
```

这个Makefile会编译SOURCES中的每个文件，然后将它们链接到一个ELF二进制文件“kernel”中。它使用链接程序脚本“link.ld”执行此操作：

### 1.5.2. Link.ld

```
/* Link.ld -- Linker script for the kernel - ensure everything goes in the */
/*            Correct place.  */
/*            Original file taken from Bran's Kernel Development */
/*            tutorials: http://www.osdever.net/bkerndev/index.php. */

ENTRY(start)
SECTIONS
{
  .text 0x100000 :
  {
    code = .; _code = .; __code = .;
    *(.text)
    . = ALIGN(4096);
  }

  .data :
  {
     data = .; _data = .; __data = .;
     *(.data)
     *(.rodata)
     . = ALIGN(4096);
  }

  .bss :
  {
    bss = .; _bss = .; __bss = .;
    *(.bss)
    . = ALIGN(4096);
  }

  end = .; _end = .; __end = .;
}
```

该脚本告诉LD如何设置我们的内核映像。首先，它告诉LD我们二进制文件的开始位置应为符号“start”。然后，它告诉LD应该将.text节（所有代码移至该节）放在第一位，并应从0x100000（1MB）开始。.data（初始化的静态数据）和.bss（未初始化的静态数据）应该紧随其后，并且每个页面都应该对齐（ALIGN（4096））。Linux GCC还添加了一个额外的数据部分：.rodata。这是只读的初始化数据，例如常量。为简单起见，我们将其与.data节捆绑在一起。

### 1.5.3. update_image.sh

一个不错的小脚本，它将您的新内核二进制文件戳入软盘映像文件（假定您已创建目录/mnt）。注意：您需要在$ PATH中使用/sbin才能使用losetup。

```shell
#!/bin/bash

sudo losetup /dev/loop0 floppy.img
sudo mount /dev/loop0 /mnt
sudo cp src/kernel /mnt/kernel
sudo umount /dev/loop0
sudo losetup -d /dev/loop0
```

### 1.5.4. run_bochs.sh

该脚本将设置loopback设备，在其上运行boch，然后断开其连接。

```shell
#!/bin/bash

# run_bochs.sh
# mounts the correct loopback device, runs bochs, then unmounts.

sudo /sbin/losetup /dev/loop0 floppy.img
sudo bochs -f bochsrc.txt
sudo /sbin/losetup -d /dev/loop0
```

# 原文

[http://www.jamesmolloy.co.uk/tutorial_html/1.-Environment%20setup.html](http://www.jamesmolloy.co.uk/tutorial_html/1.-Environment%20setup.html)
