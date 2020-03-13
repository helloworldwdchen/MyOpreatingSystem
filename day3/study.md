# 1 加快中断处理，引入FIFO缓冲区

-   加快中断处理

将按键的编码接收下来， 保存到变量里，由主函数偶尔去查看这个变量，如果有数据就把它显示出来，修改int.c ，声明一个KEYBUF结构体到bootpack.h中

```c
/* int.c */
#define PORT_KEYDAT		0x0060

struct KEYBUF keybuf;

void inthandler21(int *esp){ /* 来自PS/2键盘的中断*/
	unsigned char data;
	io_out8(PIC0_OCW2, 0x61); /* 通知PIC IRQ-01已经受理完毕*/
	data = io_in8(PORT_KEYDAT);
	if(keybuf.flag == 0){
		keybuf.data = data;
		keybuf.flag = 1;
	}
	return ;
}
```

```
/* 声明在bootpack中 */
struct KEYBUF {
	unsigned char data, flag;
};
```

flag 用来标记缓冲区是否为空，0表示空，1表示有数据。如果缓冲区里有数据而这时来了中断，就不做任何处理，当作把这个数据扔掉。

修改 bootpack.c的主函数中io_hlt的死循环

```c
extern struct KEYBUF keybuf;
for(;;){
		io_cli();
		if(keybuf.flag == 0){
			io_stihlt();
		}
		else{
			i = keybuf.data;
			keybuf.flag = 0;
			io_sti();
			sprintf(s, "%02X", i);
			boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
			putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
		}
	}
```

先用io_cil指令屏蔽中断， 防止在执行其后的处理时有中断进来。

如果flag==0， 说明键没有被按下， data没有值，我们无事可做，干脆去执行io_hlt，然而已经屏蔽了中断，如果这时候去执行HLT，即使有键按下程序也不会有任何反应，所以sti和hlt两个指令都要执行，执行这两个指令的函数就是io_stihlt。执行HLT指令以后，如果收到了PIC的通知， CPU就会被唤醒。这样， CPU首先会去执行中断处理程序。中断处理程序执行完以后，又回到for语句的开头，再执行io_cli函数。  

如果flag！=0， 先将键码存到i里，然后将flag置零，表示清空键码，然后再用sti放开中断虽然如果在keybuf操作当中有中断进来会造成混乱，但现在keybuf.data的值已经保存完毕，再开放中断也就没关系了  

然而这样写有一个**问题**，

当按下右Ctrl键时，会产生两个字节的键码值“E0 1D”，而松开这个键之后，会产生两个字节的键码值“E0 9D”。在一次产生两个字节键码值的情况下，因为键盘内部电路一次只能发送一个字节，所以一次按键就会产生两次中断，第一次中断时发送E0，第二次中断时发送1D  ，而在未加快中断之前，以上两次中断所发送的值都能收到，瞬间显示E0之后，紧接着又显示1D或是9D ，在加快后，HariMain函数在收到E0之前，又收到前一次按键产生的1D或者9D，而这个字节被舍弃了  。

-   制作FIFO缓冲区

问题到底出在哪儿呢？在于我们所创建的缓冲区，它只能存储一个字节。如果做一个能够存储多字节的缓冲区，那么它就不会马上存满，这个问题也就解决了。  

修改结构体，使data成为一个数组

```c
struct KEYBUF {
	unsigned char data[32];
	int next;
};
```

修改int.c

```c
void inthandler21(int *esp){ /* 来自PS/2键盘的中断*/
	unsigned char data;
	io_out8(PIC0_OCW2, 0x61); /* 通知PIC IRQ-01已经受理完毕*/
	data = io_in8(PORT_KEYDAT);
	if(keybuf.next<32){
		keybuf.data[keybuf.next]  = data;
		keybuf.next++;
	}
	return ;
}
```

下一个存储位置用变量next来管理。 next，就是“下一个”的意思。这样就可以记住32个数据，而不会溢出。但是为了保险起见， next的值变成32之后，就舍去不要了 。

还要修改主函数取得数据的部分

```c
for(;;){
		io_cli();
		if(keybuf.next == 0){
			io_stihlt();
		}
		else{
			i = keybuf.data[0];
			keybuf.next--;
			for(j=0;j<keybuf.next;j++){
				keybuf.data[j] = keybuf.data[j+1];
			}
			io_sti();
			sprintf(s, "%02X", i);
			boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
			putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
		}
	}
```

现在可以正常运行了

但实际上还是有问题。还有些地方还不尽如人意。inthandler21可以了，完全没有问题。有问题的是HariMain。说得具体一点，是从data[0]取得数据后有关数据移送的处理不能让人满意。像这种移送数据的处理，一般说来也就不超过3个，基本上没有什么问题。但运气不好的时候，我们可能需要移送多达32个数据。虽然这远比显示字符所需的128个像素要少，但要是有办法避免这种操作的话，当然是最好不过了  

数据移送处理本身并没有什么不好，只是在禁止中断的期间里做数据移送处理有问题。但如果在数据移送处理前就允许中断的话，会搞乱要处理的数据，这当然不行。那该怎么办才好呢？  

-   改善FIFO缓冲区

改善的方法是开发一个不需要数据移送操作的FIFO缓冲区

当下一个数据写入位置到达缓冲区最末尾时，缓冲区开头部分应该已经变空了（如
果还没有变空，说明数据读出跟不上数据写入，只能把部分数据扔掉了）。因此如果下一个数据
写入位置到了32以后，就强制性地将它设置为0。这样一来，下一个数据写入位置就跑到了下一
个数据读出位置的后面  

对下一个数据读出位置也做同样的处理，一旦到了32以后，就把它设置为从0开始继续读取
数据。这样32字节的缓冲区就能一圈一圈地不停循环  

修改结构体

```c
struct KEYBUF {
	unsigned char data[32];
	int next_r, next_w, len;
};
```

变量len是指缓冲区能记录多少字节的数据。  

修改int.c的读入部分

```c
void inthandler21(int *esp){ /* 来自PS/2键盘的中断*/
	unsigned char data;
	io_out8(PIC0_OCW2, 0x61); /* 通知PIC IRQ-01已经受理完毕*/
	data = io_in8(PORT_KEYDAT);
	if(keybuf.len<32){
		keybuf.data[keybuf.next_w] = data;
		keybuf.len++;
		keybuf.next_w++;
		if(keybuf.next_w == 32){
			keybuf.next_w = 0;
		}
	}
	return ;
}
```

修改主函数的读出部分

```c
for(;;){
		io_cli();
		if(keybuf.len == 0){
			io_stihlt();
		}
		else{
			i = keybuf.data[keybuf.next_r];
			keybuf.len--;
			keybuf.next_r++;
			if(keybuf.next_r == 32){
				keybuf.next_r = 0;
			}
			io_sti();
			sprintf(s, "%02X", i);
			boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
			putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
		}
	}
```

## 2 整理FIFO缓冲区

将FIFO缓冲区有关部分整理出来，到fifo.c

先写bootpack.h

```c
struct FIFO8{
	unsigned char *buf;
	int p, q, size, free, flags;
}
void fifo8_init(struct FIFO8 *fifo, int size, unsigned char *buf);
void fifo8_put(struct FIFO8 *fifo, unsigned char data);
int fifo8_get(struct FIFO8 *fifo);
int fifo8_status(struct FIFO8 *fifo);
```

缓冲区的总字节数保存在变量size里。变量free用于保存缓冲区里没有数据的字节数。缓冲区的地址当然也必须保存下来，我们把它保存在变量buf里。 p代表下一个数据写入地址（ next_w）， q代表下一个数据读出地址（ next_r） 

接下来写fifo.c

```c
#include "bootpack.h"

#define FLAGS_OVERRUN	0x0001
void fifo8_init(struct FIFO8 *fifo, int sise, unsigned char *buf){
	fifo->size = size;
	fifo->buf = buf;
	fifo->free = size;
	fifo->flags = 0;
	fifo->p = 0;
	fifo->q = 0;
	return ;
}

int fifo8_put(struct FIFO8 *fifo, unsigned char data){
	if(fifo->free==0){
		fifo->flags|=FLAGS_OVERRUN;
		return -1;
	}
	fifo->buf[fifo->p] = data;
	fifo->p++;
	if(fifo->p == fifo->size){
		fifo->p = 0;
	}
	fifo->free--;
	return 0;
}

int fifo8_get(struct FIFO8 *fifo){
	int data;
	if(fifo->free == fifo->size){
		return -1;
	}
	data = fifo->buff[fifo->q];
	fifo->q++;
	if(fifo->q == fifo->size){
		fifo->q = 0;
	}
	fifo->free++;
	return data;
}

int fifo8_status(struct FIFO *fifo){
	return fifo->size - fifo->free;
}
```

fifo8_put是往FIFO缓冲区存储1字节信息的函数  ,用flags这一变量来记录是否溢出。

ifo8_get是从FIFO缓冲区取出1字节的函数  

fifo8_status用来调查缓冲区的状态  

然后修改int.c

```c
struct FIFO8 keyfifo;

void inthandler21(int *esp){ /* 来自PS/2键盘的中断*/
	unsigned char data;
	io_out8(PIC0_OCW2, 0x61); /* 通知PIC IRQ-01已经受理完毕*/
	data = io_in8(PORT_KEYDAT);
	fifo8_put(&keyfifo, data);
	return ;
}
```

最后是主函数

```c
extern struct FIFO8 keyfifo;
char s[40],mcursor[256],keybuf[32];
fifo8_init(&keyfifo, 32, keybuf);

for(;;){
		io_cli();
		if(fifo8_status(&keyfifo) == 0){
			io_stihlt();
		}
		else{
			i = fifo8_get(&keyfifo);
			io_sti();
			sprintf(s, "%02X", i);
			boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
			putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
		}
	}
```

最后记得修改Makefile，加入fifo.obj