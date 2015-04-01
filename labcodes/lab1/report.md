# ================================================================================================================
# ================================================================================================================
#                                  操作系统实验 ucore Lab1 实验报告
# ================================================================================================================
# ================================================================================================================

[练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中
每一条相关命令和命令参数的含义,以及说明命令导致的结果)
# ----------------------------------------------------------------------------------------------------------------
# 分为两步，先生成bootblock，然后再生成kernel。

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
	
# --------------------------------------------------------------------
# 以下为生成bootblock的makefile代码。
# 从boot目录里找到所有与boot相关的代码，用cc_complie编译。
# 总共有asm.h，bootasm.s，bootmain.c三个文件。
bootfiles = $(call listf_cc,boot)
# 通过CFLAG的定义，我们可以得到重要的编译参数
# -ggdb可以生成供gdb调试的调试信息。-m32说明了生成程序运行于32位环境。-gstabs生成系统调用栈的调试信息。
# -nostdinc表示不是使用std库。
# -fno-stack-protector 表示没有stack protector，即保护栈防止栈溢出的代码。
# -fno-buildin表示没有buildin优化。
# -Os是常用的代码优化选项。-I后面跟着头文件的路径。
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
# 定义变量bootblock
bootblock = $(call totarget,bootblock)
# 先调用生成的obj文件，然后编译tools/sign.c代码。
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
# 这里调用的命令是ld，通过LDFLAGS的定义，我们可以得到重要的编译参数
# -m后面说明了collector的类型，-nostdlib表示不使用standard library。
# 接下来LDFLAGS外定义的参数如下
# -N设置了代码段和读写段的可读写状态，-e start制定了start入口，-Ttext 0xC700制定了代码开始地址为0xC700。
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
# objdump用来查看bin文件带有的附加信息。
# -S表示移除所有符号和重定位信息，-O binary表示输出格式为bin。
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
# objcopy命令用来拷贝bin文件。
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
$(call create_target,bootblock)

# --------------------------------------------------------------------
# 以下为生成sign的makefile代码。
# 通过HOSTCFLAGS的定义，我们可以得到重要的编译参数
# -Wall表示提示所有warning，-O2为优化选项，-g表示gdb需要使用。
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)

#---------------------------------------------------------------------
# 以下为生成kernel的makefile代码。
kernel = $(call totarget,kernel)
# 指定kernel的路径
$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)


[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?
# ----------------------------------------------------------------------------------------------------------------
大小为512 byte，末尾为55 AA。

[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。
# ----------------------------------------------------------------------------------------------------------------
可以通过输出log来查看bios代码的运行情况。查看qemu运行参数的说明，发现启动时加参数-qemu -d可以根据需要打出log。
Log items (comma separated):
out_asm   show generated host assembly code for each compiled TB
in_asm    show target assembly code for each compiled TB
在qemu -d后面加in_asm就可以就能把guest code和翻译好的host code打印出来。
所以可以通过改写makefile文件，来添加qemu运行时的参数。改写部分如下：
debug: $(UCOREIMG)
	   $(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -parallel stdio -hda $< -serial null"
	   $(V)sleep 2
	   $(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
这样在调用gdb时，会自动处理tools/gdbinit中的语句。
gdbinit中的内容如下：
file bin/kernel
target remote :1234
break kern_init

[练习2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。
# ----------------------------------------------------------------------------------------------------------------
在检查断点之前，需要将80386模式的CPU设置成8086模式，测试完毕后CPU再返回原来的状态。
qemu里有设置CPU模式的命令set architecture，8086模式是i8086，80386模式是i386。
所以gdbinit中的内容如下：
file bin/kernel
target remote :1234
break kern_init
set architecture i8086
b *0x7c00
continue
x /2i $pc
set architecture i386

[练习2.3] 在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。
将执行的汇编代码与bootasm.S 和 bootblock.asm 进行比较，看看二者是否一致。
# ----------------------------------------------------------------------------------------------------------------
将q.log中的汇编命令和bootasm.S和bootblock.asm进行比较，发现二者是相同的。

[练习3] 分析bootloader 进入保护模式的过程。
# ----------------------------------------------------------------------------------------------------------------
# 第一步，要初始化环境，将相关的寄存器初始化为0。
.code16
		# 将处理器标志寄存器的中断标志位清0，不允许中断
	    cli
		# CLD用来操作方向标志位DF。CLD使DF复位，即DF=0
	    cld
		# 将寄存器ax设为0 
	    xorw %ax, %ax
		# 将其他段寄存器初始化
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %ss
# 第二步，将A20线设为高电平，这样可以使用4G的内存空间。
seta20.1:
		# 编号为0x64的IO端口为键盘控制器端口
	    inb $0x64, %al
		# 检验控制器是否处于空闲状态
	    testb $0x2, %al
		# 若忙碌则跳转至原处，等待键盘控制器的空闲状态
	    jnz seta20.1
	
		# 发送0xd1指令给0x64的IO端口。
	    movb $0xd1, %al
	    outb %al, $0x64
	
seta20.1:
	    inb $0x64, %al
	    testb $0x2, %al
	    jnz seta20.1
	
		# 发送0xdf指令给0x64的IO端口，用来打开A20。
	    movb $0xdf, %al
	    outb %al, $0x60
# 第三步，载入引导区的gdt表
lgdt gdtdesc
# 第四步，设置保护模式
		# 将cr0寄存器的PE位设为1
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
# 第五步，通过长跳转更新cs的基地址
	    ljmp $PROT_MODE_CSEG, $protcseg
	.code32
	protcseg:
# 第六步，建立系统堆栈
		# 将段寄存器设为$PROT_MODE_DSEG
	    movw $PROT_MODE_DSEG, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %fs
	    movw %ax, %gs
	    movw %ax, %ss
		# 初始化栈顶
	    movl $0x0, %ebp
	    movl $start, %esp
# 最后，调用bootmain
	    call bootmain
	    
[练习4] 分析bootloader加载ELF格式的OS的过程。
# ----------------------------------------------------------------------------------------------------------------	   
我们主要来分析boot/bootmain.c中的代码。boot的主函数如下:
# ---------------------------------------------------------------------
void bootmain(void)
{
# 	读取ELF第一页的内容
	readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0); 
#   通过幻数，来判断ELF是否合法
	if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }
#	ph用来存放PH Table的头部地址
#	eph是PH Table的尾地址
    struct proghdr *ph, *eph;
    
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
#	读取PH Table中的每个程序头的程序
	for (; ph < eph; ph ++) 
	{
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }
#	进入进程开始的位置，系统将控制转移
	((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
}
# ---------------------------------------------------------------------
其中readseg函数的具体实现：
# ---------------------------------------------------------------------
# readseg(va, count, offset)
# va     : 存入空间的头指针
# count  : 空间的大小
# offset : 头地址的偏移量
static void 
readseg(uintptr_t va, uint32_t count, uint32_t offset) 
{
# 	空间的尾地址
    uintptr_t end_va = va + count;
#	头指针根据offset偏移
    va -= offset % SECTSIZE;
#	将offset的地址偏移转为扇区数量
    uint32_t secno = (offset / SECTSIZE) + 1;
#	分扇区读取
    for (; va < end_va; va += SECTSIZE, secno ++) 
    {
        readsect((void *)va, secno);
    }
}
# ---------------------------------------------------------------------
扇区读取函数readsect的实现：
# ---------------------------------------------------------------------
# readsect(vs, secno)
# vs    : 扇区头地址
# secno : 扇区号
readsect(void *dst, uint32_t secno) {
#	等待硬盘准备就绪
    waitdisk();
#	将扇区号写入IO端口0x1F2-0x1F7
    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors
#	等待硬盘准备就绪
    waitdisk();
#	从IO端口0x1F0读取扇区数据
    insl(0x1F0, dst, SECTSIZE / 4);
}
# ---------------------------------------------------------------------
通过IO端口0x1F7的头2位是否等于10，来判断硬盘的空闲状态。
# ---------------------------------------------------------------------
static void
waitdisk(void) 
{
    while ((inb(0x1F7) & 0xC0) != 0x40) ;
}

[练习5] 实现函数调用堆栈跟踪函数。并解释最后一行各个数值的含义。
# ----------------------------------------------------------------------------------------------------------------	
运行"make qemu"命令，获得如下输出。
……
ebp:0x00007b28 eip:0x00100992 args:0x00010094 0x00010094 0x00007b58 0x00100096
    kern/debug/kdebug.c:305: print_stackframe+22
    
ebp:0x00007b38 eip:0x00100c79 args:0x00000000 0x00000000 0x00000000 0x00007ba8
    kern/debug/kmonitor.c:125: mon_backtrace+10
    
ebp:0x00007b58 eip:0x00100096 args:0x00000000 0x00007b80 0xffff0000 0x00007b84
    kern/init/init.c:48: grade_backtrace2+33
    
ebp:0x00007b78 eip:0x001000bf args:0x00000000 0xffff0000 0x00007ba4 0x00000029
    kern/init/init.c:53: grade_backtrace1+38
    
ebp:0x00007b98 eip:0x001000dd args:0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:58: grade_backtrace0+23
    
ebp:0x00007bb8 eip:0x00100102 args:0x0010353c 0x00103520 0x00001308 0x00000000
    kern/init/init.c:63: grade_backtrace+34
    
ebp:0x00007be8 eip:0x00100059 args:0x00000000 0x00000000 0x00000000 0x00007c53
    kern/init/init.c:28: kern_init+88
    
ebp:0x00007bf8 eip:0x00007d73 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
<unknow>: -- 0x00007d72 –
……
ebp为地址所在的空间存放上一个ebp。 eip为函数调用完毕时程序的返回地址。
ebp的下一个空间保存的是上一个eip。接下来就是压入栈的参数。
最后一行的ebp为调用该进程的代码，boot的主程序bootmain。
栈底部为0x7f00，而调用call命令时原PC压入栈，所以栈顶变为0x7bf8。

[练习6.1] 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
# ----------------------------------------------------------------------------------------------------------------
中断向量的表项占8个字节，2-3字节为段选择子，0-1和6-7为位移，共同构成代码入口地址。

[练习6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。
# ----------------------------------------------------------------------------------------------------------------


[练习6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数
# ----------------------------------------------------------------------------------------------------------------


[练习7.1] 增加syscall功能，即增加一用户态函数（可执行一特定系统调用：获得时钟计数值），当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的函数又通过系统调用得到内核态的服务。
# ----------------------------------------------------------------------------------------------------------------
该功能分为二部分：
#--------------------------------------------------------------------
(1) 内核态切换到用户态。
通过trap.h，得到两种切换的中断编号。
#define T_SWITCH_TOU                120    // user/kernel switch
#define T_SWITCH_TOK                121    // user/kernel switch
在由内核态切换成用户态的时候，一开始调用中断时，由于是从内核态调用的，没有权限切换，故ss、esp没有压栈，而iret返回时，是返回到用户态，故ss、esp会出栈，于是为了保证栈的正确性，需要在调用中断前将esp减8以预留空间，中断返回后，由于esp被修改，还需要手动恢复esp为正确值。
所以lab1_switch_to_user函数的实现如下
asm volatile (
	    "sub $0x8, %%esp \n"
	    "int %0 \n"
	    "movl %%ebp, %%esp"
	    : 
	    : "i"(T_SWITCH_TOU)
);
#--------------------------------------------------------------------
(2) 用户态切换到内核态。
lab1_switch_to_kernel函数的实现如下
	asm volatile (
	    "int %0 \n"
	    "movl %%ebp, %%esp \n"
	    : 
	    : "i"(T_SWITCH_TOK)
);




