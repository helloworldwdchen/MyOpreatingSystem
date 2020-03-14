# 1 整理源文件 与解释asmhead.nas

-   整理源文件

新建mouse.c 和keyboard.c ，将鼠标键盘中断的函数从主函数和init.c中取出来

在MakeFile里添加两个文件的.obj

-   解释asmhead.nas

-   开始做的事情

```
; 	PIC关闭一切中断
;	根据AT兼容机的规格，如果要初始化PIC
;	必须在CLI之前进行，否则有时会挂起
;	随后进行PIC的初始化

		MOV		AL,0xff
		OUT		0x21,AL
		NOP						; 如果连续执行OUT指令，有些机种会无法正常运行
		OUT		0xa1,AL

		CLI						; 禁止CPU级别的中断
```

这段程序等同于以下的C程序

```c
io_out(PIC0_IMR, 0xff); /* 禁止主PIC的全部中断 */
io_out(PIC1_IMR, 0xff); /* 禁止从PIC的全部中断 */
io_cli(); /* 禁止CPU级别的中断*/
```

如果当CPU进行模式转换时进来了中断信号，那可就麻烦了。而且，后来还要进行PIC的初始化，初始化时也不允许有中断发生。所以，我们要把中断全部屏蔽掉。
顺便说一下， NOP指令什么都不做，它只是让CPU休息一个时钟长的时间  

-   接下来

```
; C为了让CPU能够访问1MB以上的内存空间，设定A20GATE

		CALL	waitkbdout
		MOV		AL,0xd1
		OUT		0x64,AL
		CALL	waitkbdout
		MOV		AL,0xdf			; enable A20
		OUT		0x60,AL
		CALL	waitkbdout
```

这里的waitkbdout，等同于wait_KBC_sendready  

相当于C语言程序：

```c
#define KEYCMD_WRITE_OUTPORT 0xd1
#define KBC_OUTPORT_A20G_ENABLE 0xdf
/* A20GATE的设定 */
wait_KBC_sendready();
io_out8(PORT_KEYCMD, KEYCMD_WRITE_OUTPORT);
wait_KBC_sendready();
io_out8(PORT_KEYDAT, KBC_OUTPORT_A20G_ENABLE);
wait_KBC_sendready(); /* 这句话是为了等待完成执行指令 */
```

程序的基本结构与init_keyboard完全相同，功能仅仅是往键盘控制电路发送指令。  

这里发送的指令，是指令键盘控制电路的附属端口输出0xdf。这个附属端口，连接着主板上的很多地方，通过这个端口发送不同的指令，就可以实现各种各样的控制功能  

这次输出0xdf所要完成的功能，是让A20GATE信号线变成ON的状态。这条信号线的作用是什么呢？它能使内存的1MB以上的部分变成可使用状态。最初出现电脑的时候， CPU只有16位模式，所以内存最大也只有1MB。后来CPU变聪明了，可以使用很大的内存了。但为了兼容旧版的操作系统，在执行激活指令之前，电路被限制为只能使用1MB内存。和鼠标的情况很类似哟。A20GATE信号线正是用来使这个电路停止从而让所有内存都可以使用的东西  

最后还有一点， “wait_KBC_sendready();”是多余的。在此之后，虽然不会往键盘送命令，
但仍然要等到下一个命令能够送来为止。这是为了等待A20GATE的处理切实完成  

-   继续往下

```
; 切换到保护模式

[INSTRSET "i486p"]				; “想要使用486指令”的叙述

		LGDT	[GDTR0]			; 设定临时GDT
		MOV		EAX,CR0
		AND		EAX,0x7fffffff	; 设bit31为0（为了禁止分页）
		OR		EAX,0x00000001	; 设bit0为1（为了切换到保护模式）
		MOV		CR0,EAX
		JMP		pipelineflush
pipelineflush:
		MOV		AX,1*8			;  可读写的段 32bit
		MOV		DS,AX
		MOV		ES,AX
		MOV		FS,AX
		MOV		GS,AX
		MOV		SS,AX
```

INSTRSET指令，是为了能够使用386以后的LGDT， EAX， CR0等关键字。  

LGDT指令，不管三七二十一，把随意准备的GDT给读进来。对于这个暂定的GDT，我们以后还要重新设置  

然后将CR0这一特殊的32位寄存器的值代入EAX，并将最高位置为0，最低位=置为1，再将这个值返回给CR0寄存器。这样就完成了模式转换，进入到不用分页的保护模式。 CR0，也就是control register 0，是一个非常重要的寄存器，只有操作系统才能操作它  。

保护模式①与先前的16位模式不同，段寄存器的解释不是16倍，而是能够使用GDT。这里的“保护”，来自英文的“protect”。在这种模式下，应用程序既不能随便改变段的设定，又不能使用操作系统专用的段。操作系统受到CPU的保护，所以称为保护模式  。

在保护模式中，有带保护的16位模式，和带保护的32位模式两种。我们要使用的，是带保护的32位模式  

讲解CPU的书上会写到，通过代入CR0而切换到保护模式时，要马上执行JMP指令。所以我们也执行这一指令。为什么要执行JMP指令呢？因为变成保护模式后，机器语言的解释要发生变化。 CPU为了加快指令的执行速度而使用了管道（ pipeline）这一机制，就是说，前一条指令还在执行的时候，就开始解释下一条甚至是再下一条指令。因为模式变了，就要重新解释一遍，所以加入了JMP指令  

而且在程序中，进入保护模式以后，段寄存器的意思也变了（不再是乘以16后再加算的意思了），除了CS以外所有段寄存器的值都从0x0000变成了0x0008。CS保持原状是因为如果CS也变了，会造成混乱，所以只有CS要放到后面再处理。 0x0008，相当于“gdt + 1”的段  

-   继续读程序

```
; bootpack的转送

		MOV		ESI,bootpack	; 转送源
		MOV		EDI,BOTPAK		; 转送目的地
		MOV		ECX,512*1024/4
		CALL	memcpy

; 磁盘数据最终转送到它本来的位置去

; 首先从启动扇区开始

		MOV		ESI,0x7c00		; 转送源
		MOV		EDI,DSKCAC		; 转送目的地
		MOV		ECX,512/4
		CALL	memcpy

; 所有剩下的

		MOV		ESI,DSKCAC0+512	; 转送源
		MOV		EDI,DSKCAC+512	;  转送目的地
		MOV		ECX,0
		MOV		CL,BYTE [CYLS]
		IMUL	ECX,512*18*2/4	; 从柱面数变换为字节数/4
		SUB		ECX,512/4		;  减去 IPL
		CALL	memcpy
```

简单来说，这部分程序只是在调用memcpy函数。

C语言写法

```c
memcpy(bootpack, BOTPAK, 512*1024/4);
memcpy(0x7c00, DSKCAC, 512/4 );
memcpy(DSKCAC0+512, DSKCAC+512, cyls * 512*18*2/4 - 512/4);
```

函数memcpy是复制内存的函数，语法如下：
memcpy(转送源地址, 转送目的地址, 转送数据的大小） ;  

转送数据大小是以双字为单位的，所以数据大小用字节数除以4来指定。在上面3个memcpy语句中，我们先来看看中间一句  `memcpy(0x7c00, DSKCAC, 512/4） ;  `

DSKCAC是0x00100000，所以上面这句话的意思就是从0x7c00复制512字节到0x00100000。这正好是将启动扇区复制到1MB以后的内存去的意思。下一个memcpy语句：  

`memcpy(DSKCAC0+512,DSKCAC+512, cyls * 512*18*2/4-512/4） ;  `

它的意思就是将始于0x00008200的磁盘内容，复制到0x00100200那里  

上文中“转送数据大小”的计算有点复杂，因为它是以柱面数来计算的，所以需要减去启动区的那一部分长度。这样始于0x00100000的内存部分，就与磁盘的内容相吻合了。顺便说一下，IMUL是乘法运算， SUB是减法运算。它们与ADD（加法）运算同属一类 。

现在我们还没说明的函数就只有有程序开始处的memcpy了。 bootpack是asmhead.nas的最后一个标签。 haribote.sys是通过asmhead.bin和bootpack.hrb连接起来而生成的（可以通过Makefile确认），所以asmhead结束的地方，紧接着串连着bootpack.hrb最前面的部分 。

```
memcpy(bootpack, BOTPAK, 512*1024/4） ;
```

这就是将bootpack.hrb复制到0x00280000号地址的处理。为什么是512KB呢？这是我们酌情考虑而决定的。内存多一些不会产生什么问题，所以这个长度要比bootpack.hrb的长度大出很多  。

-   最后的50行

```
; 必须由asmhead来完成的工作，至此全部完毕
;	以后就交由bootpack来完成

; bootpack的启动

		MOV		EBX,BOTPAK
		MOV		ECX,[EBX+16]
		ADD		ECX,3			; ECX += 3;
		SHR		ECX,2			; ECX /= 4;
		JZ		skip			; 没有要转送的东西时
		MOV		ESI,[EBX+20]	; 转送源
		ADD		ESI,EBX
		MOV		EDI,[EBX+12]	; 转送目的地
		CALL	memcpy
skip:
		MOV		ESP,[EBX+12]	; ; 栈初始值
		JMP		DWORD 2*8:0x0000001b
```

结果我们仍然只是在做memcpy。它对bootpack.hrb的header（头部内容）进行解析，将执行所必需的数据传送过去。 EBX里代入的是BOTPAK，所以值如下：  

```
[EBX + 16]......bootpack.hrb之后的第16号地址。值是0x11a8
[EBX + 20]......bootpack.hrb之后的第20号地址。值是0x10c8
[EBX + 12]......bootpack.hrb之后的第12号地址。值是0x00310000
```

上面这些值，是我们通过二进制编辑器，打开harib05d的bootpack.hrb后确认的。这些值因harib的版本不同而有所变化  

SHR指令是向右移位指令，相当于"ECX >>=2; "

JZ是条件跳转指令，来自英文jump if zero，根据前一个计算结果是否为0来决定是否跳转。在这里，根据SHR的结果，如果ECX变成了0，就跳转到skip那里去  

而最终这个memcpy到底用来做什么事情呢？它会将bootpack.hrb第0x10c8字节开始的0x11a8字节复制到0x00310000号地址去。大家可能不明白为什么要做这种处理，但这个问题，必须要等到“纸娃娃系统”的应用程序讲完之后才能讲清楚  

最后将0x310000代入到ESP里，然后用一个特别的JMP指令，将2 * 8 代入到CS里，同时移动到0x1b号地址。这里的0x1b号地址是指第2个段的0x1b号地址。第2个段的基地址是0x280000，所以实际上是从0x28001b开始执行的。这也就是bootpack.hrb的0x1b号地址。这样就开始执行bootpack.hrb了  



**我们的操作系统的内存分布图**：

```
0x00000000 - 0x000fffff : 虽然在启动中会多次使用，但之后就变空。（ 1MB）
0x00100000 - 0x00267fff : 用于保存软盘的内容。（ 1440KB）
0x00268000 - 0x0026f7ff : 空（ 30KB）
0x0026f800 - 0x0026ffff : IDT （ 2KB）
0x00270000 - 0x0027ffff : GDT （ 64KB）
0x00280000 - 0x002fffff : bootpack.hrb（ 512KB）
0x00300000 - 0x003fffff : 栈及其他（ 1MB）
0x00400000 - : 空
```

虽然没有明写，但在最初的1MB范围内，还有BIOS，VRAM等内容，也就是说并不是1MB全都空着  

从软盘读出来的东西，之所以要复制到0x00100000号以后的地址，就是因为我们意识中有这个内存分布图。同样，前几天，之所以能够确定正式版的GDT和IDT的地址，也是因为这个内存分布图  

-   继续看代码

```
waitkbdout:
		IN		 AL,0x64
		AND		 AL,0x02
		IN		 AL,0x60		; 书里有源代码里没有，空读（为了清空数据接收缓冲区中的垃圾数据）
		JNZ		waitkbdout		; AND的结果如果不是0，就跳到waitkbdout
		RET
```

这就是waitkbdout所完成的处理。基本上，如前面所说的那样，它与wait_KBC_sendready相同，但也添加了部分处理，就是从OX60号设备进行IN的处理。也就是说，如果控制器里有键盘代码，或者是已经累积了鼠标数据，就顺便把它们读取出来。JNZ与JZ相反，意思是“jump if not zero”  

-   接下来是memcpy程序

```
memcpy:
		MOV		EAX,[ESI]
		ADD		ESI,4
		MOV		[EDI],EAX
		ADD		EDI,4
		SUB		ECX,1
		JNZ		memcpy			; 减法运算的结果如果不是0，就跳转到memcpy
		RET
;	如果memcpy不忘记加入地址大小前缀，则可以写出字符串命令。
```

-   最后的一点内容

```
		ALIGNB	16
GDT0:
		RESB	8				; NULL selector
		DW		0xffff,0x0000,0x9200,0x00cf ; 可以读写的段（ segment） 32bit
		DW		0xffff,0x0000,0x9a28,0x0047 ; 可以执行的段（ segment） 32bit（ bootpack用）

		DW		0
GDTR0:
		DW		8*3-1
		DD		GDT0

		ALIGNB	16
bootpack:
```

ALIGNB指令的意思是，一直添加DBO，直到时机合适的时候为止。什么是“时机合适”呢？大家可能有点不明白。 ALIGNB 16的情况下，地址能被16整除的时候，就称为“时机合适”。如果最初的地址能被16整除，则ALIGNB指令不作任何处理  

如果标签GDT0的地址不是8的整数倍，向段寄存器复制的MOV指令就会慢一些。所以我们插入了ALIGNB指令。但是如果这样，“ALIGNB 8”就够了，用“ALIGNB 16”有点过头了。最后的“bootpack:”之前，也是“时机合适”的状态，所以笔者就适当加了一句“ALIGNB 16”。  

GDT0也是一种特定的GDT。 0号是空区域（ null sector），不能够在那里定义段。 1号和2号分别由下式设定  

```
set_segmdesc(gdt + 1, 0xffffffff, 0x00000000, AR_DATA32_RW);
set_segmdesc(gdt + 2, LIMIT_BOTPAK, ADR_BOTPAK, AR_CODE32_ER);
```

我们用纸笔事先计算了一下，然后用DW排列了出来。GDTR0是LGDT指令，意思是通知GDT0说“有了GDT哟”。在GDT0里，写入了16位的段上限，和32位的段起始地址  。

-   总结

到此为止，关于asmhead.nas的说明就结束了。就是说，最初状态时， GDT在asmhead.nas里，并不在0x00270000 ~ 0x0027ffff的范围里。 IDT连设定都没设定，所以仍处于中断禁止的状态。应当趁着硬件上积累过多数据而产生误动作之前，尽快开放中断，接收数据 。

因此，在bootpack.c的HariMain里，应该在进行调色板（ palette）的初始化以及画面的准备之
前，先赶紧重新创建GDT和IDT，初始化PIC，并执行“io_sti();”。  

# 2. 内存管理

1.   内容容量检查

现在我们要进行内存管理了。首先必须要做的事情，是搞清楚内存究竟有多大，范围是到哪里。如果连这一点都搞不清楚的话，内存管理就无从谈起。
在最初启动时， BIOS肯定要检查内存容量，所以只要我们问一问BIOS，就能知道内存容量有多大。但问题是，如果那样做的话，一方面asmhead.nas会变长，另一方面， BIOS版本不同， BIOS函数的调用方法也不相同，麻烦事太多了。所以，笔者想与其如此，不如自己去检查内存。  

首先，暂时让486以后的CPU的高速缓存（ cache）功能无效。回忆一下最初讲的CPU与内存的关系吧。我们说过，内存与CPU的距离地与CPU内部元件要远得多，因此在寄存器内部MOV，要比从寄存器MOV到内存快得多。但另一方面，有一个问题， CPU的记忆力太差了，即使知道内存的速度不行，还不得不频繁使用内存  

考虑到这个问题，英特尔的大叔们在CPU里也加进了一点存储器，它被称为高速缓冲存储器（ cache memory）。 cache这个词原是指储存粮食弹药等物资的仓库。但是能够跟得上CPU速度的高速存储器价格特别高，一个芯片就有一个CPU那么贵。如果128MB内存全部都用这种高价存储器，预算上肯定受不了。高速缓存，容量只有这个数值的千分之一，也就是128KB左右。高级CPU，也许能有1MB高速缓存，但即便这样，也不过就是128MB的百分之一。  

每次访问内存，都要将所访问的地址和内容存入到高速缓存里。也就是存放成这样： 18号地址的值是54。如果下次再要用18号地址的内容， CPU就不再读内存了，而是使用高速缓存的信息，马上就能回答出18号地址的内容是54  

往内存里写入数据时也一样，首先更新高速缓存的信息，然后再写入内存。如果先写入内存的话，在等待写入完成的期间， CPU处于空闲状态，这样就会影响速度。所以，先更新缓存，缓存控制电路配合内存的速度，然后再慢慢发送内存写入命令  

观察机器语言的流程会发现， 9成以上的时间耗费在循环上。所谓循环，是指程序在同一个地方来回打转。所以，那个地方的内存要一遍又一遍读进来。从第2圈循环开始，那个地方的内存信息已经保存到缓存里了，就不需要执行费时的读取内存操作了，机器语言的执行速度因而得
以大幅提高  

另外，就算是变量，也会有像“for(i = 0; i < 100; i++){}”这样， i频繁地被引用，被赋值的情况，最初是０，紧接着是１，下一个就是２。也就是说，要往内存的同一个地址，一次又一次写入不同的值。缓存控制电路观察会这一特性，在写入值不断变化的时候，试图不写入缓慢的内存，而是尽量在缓存内处理。循环处理完成，最终i的值变成100以后，才发送内存写入命令。这样，就省略了99次内存写入命令， CPU几乎不用等就能连续执行机器语言  

内存检查时，要往内存里随便写入一个值，然后马上读取，来检查读取的值与写入的值是否相等。如果内存连接正常，则写入的值能够记在内存里。如果没连接上，则读出的值肯定是乱七八糟的。方法很简单。但是，如果CPU里加上了缓存会怎么样呢？写入和读出的不是内存，而是缓存。结果，所有的内存都“正常”，检查处理不能完成  

所以，只有在内存检查时才将缓存设为OFF。具体来说，就是先查查CPU是不是在486以上，如果是，就将缓存设为OFF。按照这一思路，我们创建了以下函数memtest  

```c

#define EFLAGS_AC_BIT		0x00040000
#define CR0_CACHE_DISABLE	0x60000000

unsigned int memtest(unsigned int start, unsigned int end){
	char flg486 = 0;
	unsigned int eflg, cr0, i;

	/* 确认CPU是386还是486以上的 */
	eflg  = io_load_eflags();
	eflg |= EFLAGS_AC_BIT; 		/* AC-bit = 1 */
	io_store_eflags(eflg);
	eflg = io_load_eflags();
	if((eflg&EFLAGS_AC_BIT)!=0){	/* 如果是386，即使设定AC=1， AC的值还会自动回到0 */
		flg486 = 1;
	}
	eflg &= ~EFLAGS_AC_BIT;		/* AC-bit = 0 */
	io_store_eflags(eflg);

	if(flg486!=0){
		cr0 = load_cr0();
		cr0 |= CR0_CACHE_DISABLE;		/* 禁止缓存 */
		store_cr0(cr0);
	}

	i = memtest_sub(start, end);

	if(flg486 != 0){
		cr0 = load_cr0();
		cr0 &= ~CR0_CACHE_DISABLE;		 /* 允许缓存 */
		store_cr0(cr0);
	}

	return i;
}
```

为了禁止缓存，需要对CR0寄存器的某一标志位进行操作。对哪里操作，怎么操作，大家一看程序就能明白。这时，需要用到函数load_cr0和store_cr0，与之前的情况一样，这两个函数不能用C语言写，只能用汇编语言来写，存在naskfunc.nas里  

修改naskfunc.nas

```
_load_cr0:		; int load_cr0(void);
		MOV		EAX, CR0
		RET

_store_cr0:		; void store_cr0(int cr0);
		MOV 	EAX, [ESP+4]
		MOV		CR0, EAX
		RET
```

memtest_sub函数，是内存检查处理的实现部分  

调查从start地址到end地址的范围内，能够使用的内存的末尾地址。要做的事情很简单。首先如果p不是指针，就不能指定地址去读取内存，所以先执行“p=i;”。紧接着使用这个p，将原值保存下来（变量old）。接着试写0xaa55aa55，在内存里反转该值，检查结果是否正确①。如果正确，就再次反转它，检查一下是否能回复到初始值。最后，使用old变量，将内存的值恢复回去。……如果在某个环节没能恢复成预想的值，那么就在那个环节终止调查，并报告终止时的地址。  

i 每次增加0x1000，相当于4KB，这样一来速度比较快。 p的赋值计算式“p=i + 0xffc;”，让它只检查末尾的4个字节  

```c
unsigned int memtest_sub(unsigned int start, unsigned int end){
	unsigned int i, *p, old, pat0 = 0xaa55aa55, pat1 = 0x55aa55aa;
	for(i = start;i<=end;i+=0x1000){
		p = (unsigned int *)(i+0xffc);
		old = *p;
		*p = pat0;
		*p ^= 0xffffffff;
		if(*p != pat1){
not_memory:
			*p = old;
			break;
		}
		*p ^= 0xffffffff;
		if(*p!=pat0){
			goto not_memory;
		}
		*p = old;
	}
	return i;
}
```

再加上主函数

```c
i = memset(0x00400000, 0xbfffffff) / (1024*1024);
	sprintf(s, "memory %dMB", i);
	putfonts8_asc(binfo->vram, binfo->scrnx, 0, 32, COL8_FFFFFF, s);
```



结果运行后 显示memory有3072MB，也就是3G，然而实际上内存是32MB

原因是编译器在运行时优化掉了*p反转的过程，经过优化后程序变成了

```c
unsigned int memtest_sub(unsigned int start, unsigned int end)
{
	unsigned int i;
	for (i = start; i <= end; i += 0x1000) { }
	return i;
}
```

用于应用程序的C编译器，根本想不到会对没有内存的地方进行读写。  

于是决定memtest_sub也用汇编来写算了  

代码见naskfunc.nas

-   内存显示正常，可以回到内存管理这个正题上了

什么是内存管理呢？为什么要进
行内存管理呢?  

比如说，假设内存大小是128MB，应用程序A暂时需要100KB，画面控制需要1.2MB……，像这样，操作系统在工作中，有时需要分配一定大小的内存，用完以后又不再需要，这种事会频繁发生。为了应付这些需求，必须恰当管理好哪些内存可以使用（哪些内存空闲），哪些内存不可以使用（正在使用），这就是内存管理。如果不进行管理，系统会变得一塌糊涂，要么不知道哪里可用，要么多个应用程序使用同一地址的内存  

内存管理的基础，一是内存分配，一是内存释放。“现在要启动应用程序B了，需要84KB内存，哪儿空着呢？”如果问内存管理程序这么一个问题，内存管理程序就会给出一个能够自由使用的84KB的内存地址，这就是内存分配。另一方面，“内存使用完了，现在把内存归还给内存管理程序”，这一过程就是内存的释放过程  

我们用结构体数组来记录管理内存的信息

具体代码见bootpack.c

-   整理

将有关内存管理的部分整理出来，创建memory.c

当然也要修改Makefile和bootpack.h

为了以后使用起来更加方便，我们还是把这些内存管理函数再整理一下。 memman_alloc和memman_free能够以1字节为单位进行内存管理，这种方式虽然不错，但是有一点不足——在反复进行内存分配和内存释放之后，内存中就会出现很多不连续的小段未使用空间，这样就会把man->frees消耗殆尽  

因此，我们要编写一些总是以0x1000字节为单位进行内存分配和释放的函数，它们会把指定的内存大小按0x1000字节为单位向上舍入（ roundup），而之所以要以0x1000字节为单位，是因为笔者觉得这个数比较规整。另外， 0x1000字节的大小正好是4KB  

```c
unsigned int memman_alloc_4k(struct MEMMAN *man, unsigned int size){
	unsigned int a;
	size = (size+0xfff)&0xfffff000;
	a = memman_alloc(man, size);
	return a;
}

int memman_free_4k(struct MEMMAN *man, unsigned int addr, unsigned int size){
	int i;
	size = (size+0xfff)&0xfffff000;
	i = memman_free(man, addr, size);
	return i;
}
```

精髓是向上舍入