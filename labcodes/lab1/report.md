练习1：理解通过make生成执行文件的过程。
【1.1】ucore.img是如何一步一步生成的？
makefile中生成ucore.img的相关代码为
# create ucore.img
UCOREIMG        := $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
        $(V)dd if=/dev/zero of=$@ count=10000
        $(V)dd if=$(bootblock) of=$@ conv=notrunc
        $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
要生成ucore.img，首先需要生成kernel，bootlock。
-------------------------------------------------------------------------------------
生成kernel的makefile代码如下：
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
        @echo + ld $@
        $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
        @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
        @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
使用make "V="来查看具体的过程，如下：
+ ld bin/kernel
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o

为了生成kernel，需要kernel.ld,init.o,readline.o等等。其中kernel.ld已经存在。生成其他*.o文件的参数均类似，以init.o为例，如下：
+ cc kern/init/init.c
gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o

最终生成kernel的命令为：
+ ld bin/kernel
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
-------------------------------------------------------------------------------
生成bootlock的makefile代码为：
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

$(call create_target,bootblock)
具体的执行命令为：
+ ld bin/bootblock
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
为了生成bootblock，首先需要生成bootasm.o, bootmain.o, sign。
生成bootasm.o的命令如下：
+ cc boot/bootasm.S
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
生成bootmain.o的命令如下:
+ cc boot/bootmain.c
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
上述命令中，-ggdb用于生成gdb使用的调试信息；-m32用于生成32位环境的代码；-gstabs生成stabs格式的调试信息；-nostdinc用于说明不适用标准库；-fno-stack-protector用于说明不生成用于检测缓冲区溢出的代码；-Os是优化选项。
生成sign工具的makefile代码为：
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
具体命令如下：
+ cc tools/sign.c
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
要链接生成bootblock，还需要生成bootblock.o，命令为：
+ ld bin/bootblock
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
然后拷贝二进制代码bootblock.o到bootblock.out。
最后生成一个有10000个block的文件，每个block默认512bytes，用0填充。
dd if=/dev/zero of=bin/ucore.img count=10000
把bootblock中的内容写到第一个block：
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
从第二个block开始写kernel的内容：
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc

【1.2】一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
根据tools/sign.c，一个硬盘主引导区只有512bytes，且第510（倒数第二个）字节是0x55，最后一个是0xAA。

【练习2】使用qemu执行并调试lab1中的软件
【2.1】从CPU加电后执行的第一条指令开始,单步跟踪BIOS的执行。
在lab1目录下，执行
make debug
可以看到gdb的调试界面，此时系统停在kern_init()处，这是因为tools/gdbinit文件中在kern_init处下了一个断点，若想从第一条指令开始，则删掉gdbinit文件中的
break kern_init
continue
进入gdb调试界面后使用si可以进行单步跟踪。
可以使用x /2i $pc显示当前eip处的汇编指令。

【2.2】在初始化位置0x7c00设置实地址断点,测试断点正常。
修改gdbinit文件，在0x7c00设置断点，代码如下：
b *0x7c00
continue
为了显示当前汇编指令，添加如下指令：
x /2i $pc
运行make debug，发现确实停在了0x00007c00处：
remote Thread 1 In:                                       Line: ??   PC: 0x7c00 
0x0000fff0 in ?? ()
Breakpoint 1 at 0x7c00

Breakpoint 1, 0x00007c00 in ?? ()
=> 0x7c00:      cli    
   0x7c01:      cld 



【2.3】从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和	bootblock.asm进行比较。
（参考答案）修改Makefile，在调用qemu时增加 ’-d in_asm -D q.log‘ 参数，便可以将运行的汇编指令保存在q.log中。发现刚开始记录的地址为0xfffffff0，如下：
----------------
IN:
0xfffffff0:  ljmp   $0xf000,$0xe05b

----------------
IN:
0x000fe05b:  cmpl   $0x0,%cs:0x65a4
0x000fe062:  jne    0xfd2b9

----------------
IN:
0x000fe066:  xor    %ax,%ax
0x000fe068:  mov    %ax,%ss
到地址0x00007c00时，代码为：
----------------
IN:
0x00007c00:  cli

----------------
IN:
0x00007c00:  cli

----------------
IN:
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
……
和bootasm.S和bootblock.asm中的汇编代码相同。


【2.4】自己找一个bootloader或内核中的代码位置,设置断点并进行测试。
可以通过在gdbinit文件中对需要设断点的函数名称或者地址设断点，进行调试。

【练习3】分析bootloader进入保护模式的过程。
BIOS从磁盘的第一个扇区加载执行代码至内存0x7c00处，并开始以实模式执行。
首先，关中断，清除方向标志位，将段寄存器置0，如下：
.code16
	cli
	cld
	
	xorw %ax, %ax
	movw %ax, %ds
	movw %ax, %es
	movw %ax, %ss
开启A20:打开后可以使用2^32的物理地址空间。具体步骤为：
1）等待8042 Input buffer为空（不忙）
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1
2）发送写8042输出端口的指令
movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
3）等待8042 Input buffer为空（不忙）
seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2
4）打开A20
movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
初始化GDT表：一个简单的GDT表和其描述符已经惊天存储在引导区中，直接载入即可。
lgdt gdtdesc
将cr0寄存器PE位置1，开启保护模式：
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
使用长跳转，进入32bit模式：
ljmp $PROT_MODE_CSEG, $protcseg

.code32                                             # Assemble for 32-bit mode
protcseg:
设置保护模据段寄存器：
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment
建立堆栈：
    movl $0x0, %ebp
    movl $start, %esp
转到保护模式完成，进入bootmain:
    call bootmain

【练习4】分析bootloader加载ELF格式的OS的过程。
加载ELF格式的OS在bootmain()内完成，首先是readseg()函数，用于读取ELF的头部，然后根据ELFHDR的参数magic是否是ELF_MAGIC来判断是否是合法的ELF文件；然后加载各程序段；根据ELF文件头中入点信息，找到执行文件的入口，开始执行。
其中读取ELF中各程序段使用了readseg()函数，如下：
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // round down to sector boundary
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    uint32_t secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
	readsect((void *)va, secno);
    }
}
该函数主要是封装使用了readsect函数，实现了可以从设备读取任意长度的内容到某虚拟地址。
readsect()函数如下：
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
该函数实现了从设备的secno位置读取1个扇区数据到dst位置。

【练习5】实现函数调用堆栈跟踪函数。
实现：
ss:[ebp]存储这调用函数的ebp，因此可以得到调用栈中所有函数的ebp。另外，ss:[ebp+4]指向函数调用时的eip。
1.使用read_ebp()和read_eip读取ebp。
2.这里直接规定了STACKFRAME_DEPTH是20。依次遍历栈帧，按照输出格式，打印出ebp和eip。ebp+2处的4个参数；
3.换行，打印debuginfo。
最后一行，即最深一层为：
ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
	<unknow>: -- 0x00007d67



【练习6】完善中断初始化和处理
[6.1]中断描述符表中的一个表项占多少字节？其中哪几位代表中断处理代码的入口？
中断向量表的一个表项占8个字节，其中2~3字节是段选择子，0~1和6~7字节拼成位移，二者组成中断 处理代码入口地址。

[6.2]请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。
见代码

[练习6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数
见代码

【扩展练习1】
加syscall功能，即增加一用户态函数（可执行一特定>系统调用：获得时钟计数值），
当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的>函数又通过系统调用得到内核态的服务
在idt_init中，将用户态调用SWITCH_TOK中断权限打开。
	SETGATE(idt[T_SWITCH_TOK],1,KERNEL_CS, __vectors[T_SWTICH_TOK],3);

在init.c中，分别调用两个函数，触发中断T_SWITCH_TOU和T_SWITCH_TOK。
	asm volatile ( 
			"sub $0x8, %%esp \n"
	        "int %0 \n"
			"movl %%ebp, %%esp"
			:
			: "i"(T_SWITCH_TOU)
	);
调用T_SWITCH_TOU中断返回时，会多pop两位，并用这两位的值更新ss,sp，损坏堆栈。多以要先把栈压两位，并在中断返回后修复esp。
	asm volatile (
			"sub $0x8, %%esp \n"
	        "int %0 \n"
			"movl %%ebp, %%esp"
			:
			: "i"(T_SWITCH_TOU)
	);
调用T_SWITCH_TOK返回时，esp仍在TSS指示的堆栈中。所以在中断返回后要修复esp。
在trap_dispatch中对中断进行具体的响应，即对从堆栈中弹出的段寄存器进行修改。
为了正常输出文本，需要在trap_dispatch中转USER时，将调用io所需权限降低。
	tf->tf_eflags |= 0x3000;

