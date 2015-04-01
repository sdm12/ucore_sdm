# Lab1 report

## [练习1]

### [练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果)

> 生成ucore.img相关代码为：

 ```
 # create ucore.img
 UCOREIMG	:= $(call totarget,ucore.img)
 $(UCOREIMG): $(kernel) $(bootblock)
 	$(V)dd if=/dev/zero of=$@ count=10000
 	$(V)dd if=$(bootblock) of=$@ conv=notrunc
 	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
 ```
 
> 生成ucore.img需要先生成kernel和bootblock
 
> 生成kernel的makefile相关代码为
 ```
 # create kernel target
 kernel = $(call totarget,kernel)
 $(kernel): tools/kernel.ld
 $(kernel): $(KOBJS)
 	@echo + ld $@
 	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
 	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
 	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' >$(callsymfile,kernel)
 ```
 
> 为了生成kernel，首先需要 kernel.ld init.o readline.o stdio.o kdebug.o
	kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o
	trapentry.o vectors.o pmm.o  printfmt.o string.o

> 生成这些.o文件的相关makefile代码为
 
 ```
	$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,\
	$(KCFLAGS))
 ```
> 生成这些.o文件的方法均类似，以init.o为例：
 编译需要init.c
 实际命令为
 
 ```
	gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 \
	-gstabs -nostdinc  -fno-stack-protector \
	-Ilibs/ -Ikern/debug/ -Ikern/driver/ \
	-Ikern/trap/ -Ikern/mm/ -c kern/init/init.c \
	-o obj/kern/init/init.o
 ```
 
> 生成kernel时，makefile的几条指令中有@前缀的都不必需
	| 必需的命令只有
	| ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
	| 	obj/kern/init/init.o obj/kern/libs/readline.o \
	| 	obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
	| 	obj/kern/debug/kmonitor.o obj/kern/debug/panic.o \
	| 	obj/kern/driver/clock.o obj/kern/driver/console.o \
	| 	obj/kern/driver/intr.o obj/kern/driver/picirq.o \
	| 	obj/kern/trap/trap.o obj/kern/trap/trapentry.o \
	| 	obj/kern/trap/vectors.o obj/kern/mm/pmm.o \
	| 	obj/libs/printfmt.o obj/libs/string.o

> 生成bootblock的makefile相关代码为

```
 # create bootblock
 bootfiles = $(call listf_cc,boot)
 $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

 bootblock = $(call totarget,bootblock)

 $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

> 为了生成bootblock，首先需要生成bootasm.o、bootmain.o、sign

> 生成bootasm.o,bootmain.o的相关makefile代码为
	| bootfiles = $(call listf_cc,boot) 
	| $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),\
	|	$(CFLAGS) -Os -nostdinc))
	| 实际代码由宏批量生成
	| 
	| 生成bootasm.o需要bootasm.S
	| 实际命令为
	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \
	| 	-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \
	| 	-c boot/bootasm.S -o obj/boot/bootasm.o
 	|生成bootmain.o需要bootmain.c
	| 实际命令为
	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc \
	| 	-fno-stack-protector -Ilibs/ -Os -nostdinc \
	| 	-c boot/bootmain.c -o obj/boot/bootmain.o

> 生成sign工具的makefile代码为
	| $(call add_files_host,tools/sign.c,sign,sign)
	| $(call create_target_host,sign,sign)
	| 
	| 实际命令为
	| gcc -Itools/ -g -Wall -O2 -c tools/sign.c \
	| 	-o obj/sign/tools/sign.o
	| gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign

> 生成bootblock.o
	| ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
	|	obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
	| 拷贝二进制代码bootblock.o到bootblock.out
	| objcopy -S -O binary obj/bootblock.o obj/bootblock.out
	| 使用sign工具处理bootblock.out，生成bootblock
	| bin/sign obj/bootblock.out bin/bootblock
 生成一个有10000个块的文件，每个块默认512字节，用0填充
 dd if=/dev/zero of=bin/ucore.img count=10000

> 把bootblock中的内容写到第一个块
> dd if=bin/bootblock of=bin/ucore.img conv=notrunc

> 从第二个块开始写kernel中的内容
> dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc

###[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

> 从sign.c的代码来看，一个磁盘主引导扇区只有512字节。且第510个（倒数第二个）字节是0x55，
第511个（倒数第一个）字节是0xAA。

##[练习2]

###[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

> 在make debug后，启动qemu，按ctrl+alt+2可以进入qemu控制台，可以在那里查看qemu实际执行的BIOS汇编指令，在gdb中nexti进行单步跟踪。

###[练习2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。
> 将gdbinit文件改为如下内容即可：
	set arch i8086 
	target remote :1234
	b *0x7c00
	b *0x7c32
	set arch i386
运行"make debug"可得到
	Breakpoint 1, 0x00007c00 in ?? ()
	=> 0x7c00:      cli    
	   0x7c01:      cld    
	   0x7c02:      xor    %eax,%eax
	   0x7c04:      mov    %eax,%ds
	   0x7c06:      mov    %eax,%es
	   0x7c08:      mov    %eax,%ss 
	   0x7c0a:      in     $0x64,%al
	   0x7c0c:      test   $0x2,%al
	   0x7c0e:      jne    0x7c0a
	   0x7c10:      mov    $0xd1,%al

###[练习2.3] 在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。
将执行的汇编代码与bootasm.S 和 bootblock.asm 进行比较，看看二者是否一致。

> 将tools/gdbinit改为如下内容：
	set arch i8086 
	target remote :1234
	b *0x7c4a
	c
查看q.log后得到

```
	IN: 
	0x00007c00:  cli    
	0x00007c01:  cld    
	0x00007c02:  xor    %ax,%ax
	0x00007c04:  mov    %ax,%ds
	0x00007c06:  mov    %ax,%es
	0x00007c08:  mov    %ax,%ss

	----------------
	IN: 
	0x00007c0a:  in     $0x64,%al

	----------------
	IN: 
	0x00007c0c:  test   $0x2,%al
	0x00007c0e:  jne    0x7c0a

	----------------
	IN: 
	0x00007c10:  mov    $0xd1,%al
	0x00007c12:  out    %al,$0x64
	0x00007c14:  in     $0x64,%al
	0x00007c16:  test   $0x2,%al
	0x00007c18:  jne    0x7c14

	----------------
	IN: 
	0x00007c1a:  mov    $0xdf,%al
	0x00007c1c:  out    %al,$0x60
	0x00007c1e:  lgdtw  0x7c6c
	0x00007c23:  mov    %cr0,%eax
	0x00007c26:  or     $0x1,%eax
	0x00007c2a:  mov    %eax,%cr0

	----------------
	IN: 
	0x00007c2d:  ljmp   $0x8,$0x7c32

	----------------
	IN: 
	0x00007c32:  mov    $0x10,%ax
	0x00007c36:  mov    %eax,%ds

	----------------
	IN: 
	0x00007c38:  mov    %eax,%es

	----------------
	IN: 
	0x00007c3a:  mov    %eax,%fs
	0x00007c3c:  mov    %eax,%gs
	0x00007c3e:  mov    %eax,%ss

	----------------
	IN: 
	0x00007c40:  mov    $0x0,%ebp

	----------------
	IN: 
	0x00007c45:  mov    $0x7c00,%esp
	0x00007c4a:  call   0x7cd1
```

> 与bootasm.S比较后发现是一致的

##[练习3] 分析bootloader 进入保护模式的过程。

> 通过读bootasm.s的代码可以知道：
 从%cs=0 $pc=0x7c00，进入后
 首先清理环境：包括将flag置0和将段寄存器置0
	.code16
	    cli
	    cld
	    xorw %ax, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %ss

> 开启A20：通过将键盘控制器上的A20线置于高电位，全部32条地址线可用，
 可以访问4G的内存空间。
	seta20.1:               # 等待8042键盘控制器不忙
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.1        #
	    movb $0xd1, %al     # 发送写8042输出端口的指令
	    outb %al, $0x64     #
	seta20.1:               # 等待8042键盘控制器不忙
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.1        #
	    movb $0xdf, %al     # 打开A20
	    outb %al, $0x60     # 
初始化GDT表：一个简单的GDT表和其描述符已经静态储存在引导区中，载入即可
	    lgdt gdtdesc
进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
通过长跳转更新cs的基地址
	    ljmp $PROT_MODE_CSEG, $protcseg
	.code32
	protcseg:
设置段寄存器，并建立堆栈
	    movw $PROT_MODE_DSEG, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %fs
	    movw %ax, %gs
	    movw %ax, %ss
	    movl $0x0, %ebp
	    movl $start, %esp
转到保护模式完成，进入boot主方法
	    call bootmain

##[练习4] ：分析bootloader加载ELF格式的OS的过程。
> bootloader通过readsect和readdesc函数进行硬盘的读写
 首先看readsect函数，
 readsect从设备的第secno扇区读取数据到dst位置

```
	static void
	readsect(void *dst, uint32_t secno) {
	    waitdisk();
	
	    outb(0x1F2, 1);                         // 设置读取扇区的数目为1
	    outb(0x1F3, secno & 0xFF);
	    outb(0x1F4, (secno >> 8) & 0xFF);
	    outb(0x1F5, (secno >> 16) & 0xFF);
	    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	        // 上面四条指令联合制定了扇区号
	        // 在这4个字节线联合构成的32位参数中
	        //   29-31位强制设为1
	        //   28位(=0)表示访问"Disk 0"
	        //   0-27位是28位的偏移量
	    outb(0x1F7, 0x20);                      // 0x20命令，读取扇区
	
	    waitdisk();

	    insl(0x1F0, dst, SECTSIZE / 4);         // 读取到dst位置，
	                                            // 幻数4因为这里以DW为单位
	}
```

> readseg简单包装了readsect，可以从设备读取任意长度的内容。

```
	static void
	readseg(uintptr_t va, uint32_t count, uint32_t offset) {
	    uintptr_t end_va = va + count;
	
	    va -= offset % SECTSIZE;
	
	    uint32_t secno = (offset / SECTSIZE) + 1; 
	    // 加1因为0扇区被引导占用
	    // ELF文件从1扇区开始
	
	    for (; va < end_va; va += SECTSIZE, secno ++) {
	        readsect((void *)va, secno);
	    }
	}
```

> bootloader通过bootmain加载ELF格式的OS
 在bootmain函数中，

```
	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```

##[练习5] 实现函数调用堆栈跟踪函数 

> 输出中，堆栈最深一层为

```
	ebp:0x00007bf8 eip:0x00007d68 \
		args:0x00000000 0x00000000 0x00000000 0x00007c4f
	    <unknow>: -- 0x00007d67 --
```

> 其对应的是第一个使用堆栈的函数，bootmain.c中的bootmain。
 bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。
 call指令压栈，所以bootmain中ebp为0x7bf8。

##[练习6] 完善中断初始化和处理

###[练习6.1] 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

> 中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，
 两者联合便是中断处理程序的入口地址。

### [练习6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。
> 见代码

### [练习6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数
> 见代码
