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

# 3 增加鼠标中断，获得鼠标中断信息

-   增加鼠标中断

为了使用鼠标，必须让鼠标控制电路和鼠标本身有效，要先让鼠标控制电路有效，再让鼠标有效。

现在说说控制电路的设定。事实上，鼠标控制电路包含在键盘控制电路里，如果键盘控制电路的初始化正常完成，鼠标电路控制器的激活也就完成了。

修改bootpack.c

```c

void enable_mouse(void);
void init_keyboard(void);

#define PORT_KEYDAT 0x0060
#define PORT_KEYSTA 0x0064
#define PORT_KEYCMD 0x0064
#define KEYSTA_SEND_NOTREADY 0x02
#define KEYCMD_WRITE_MODE 0x60
#define KBC_MODE 0x47

void wait_KBC_sendready(void){
	for(;;){
		if((io_in8(PORT_KEYSTA)&KEYSTA_SEND_NOTREADY)==0)break;
	}
	return ;
}

void init_keyboard(void){
	wait_KBC_sendready();
	io_out8(PORT_KEYCMD, KEYCMD_WRITE_MODE);
	wait_KBC_sendready();
	io_out8(PORT_KEYDAT, KBC_MODE);
	return ;
}

#define KEYCMD_SENDTO_MOUSE		0xd4
#define MOUSECMD_ENABLE			0xf4

void enable_mouse(void){
	wait_KBC_sendready();
	io_out8(PORT_KEYCMD, KEYCMD_SENDTO_MOUSE);
	wait_KBC_sendready();
	io_out8(PORT_KEYDAT, MOUSECMD_ENABLE);
}
```

wait_KBC_sendready的作用是，让键盘控制电路（ keyboard controller,KBC）做好准备动作，等待控制指令的到来。为什么要做这个工作呢？是因为虽然CPU的电路很快，但键盘控制电路却没有那么快。如果CPU不顾设备接收数据的能力，只是一个劲儿地发指令的话，有些指令会得不到执行，从而导致错误的结果 。

如果键盘控制电路可以接受CPU指令了，CPU从设备号码0x0064处所读取的数据的倒数第二位（从低位开始数的第二位）应该是0。在确认到这一位是0之前，程序一直通过for语句循环查询 。

init_keyboard。它所要完成的工作很简单，也就是一边确认可否往键盘控制电路传送信息，一边发送模式设定指令，指令中包含着要设定为何种模式。模式设定的指令是0x60，利用鼠标模式的模式号码是0x47，当然这些数值必须通过调查才能知道。我们可以从http://community.osdev.info/?ifno得到这些数据

在HariMain函数调用init_keyboard函数，鼠标控制电路的准备就完成了  

我们开始发送激活鼠标的指令。所谓发送鼠标激活指令，归根到底还是要向键盘控制器发送指令  

enable_mouse  与init_keyboard函数非常相似。不同点仅在于写入的数据不同。如果往键盘控制电
路发送指令0xd4，下一个数据就会自动发送给鼠标。我们根据这一特性来发送激活鼠标的指令  

鼠标收到激活指令后， 给CPU发送信息：从现在开始就要不停地发送鼠标信息 ，这个信息就是0xfa  

-   从鼠标接受数据

现在取出中断信息

修改int.c 中鼠标中断的部分

```c
struct FIFO8 mousefifo;

void inthandler2c(int *esp){ /* 来自PS/2鼠标的中断*/
	unsigned char data;
	io_out8(PIC1_OCW2, 0x64);
	io_out8(PIC0_OCW2, 0x62);
	data = io_out8(PORT_KEYDAT);
	fifo8_put(&mousefifo, data);
	return ;
}
```

与键盘中断的不同之处只有送给PIC的中断受理通知。 IRQ-12是从PIC的第4号（从PIC相当于IRQ-08～
IRQ-15），首先要通知IRQ-12受理已完成，然后再通知主PIC。这是因为主/从PIC的协调不能够自动完成，如果程序不教给主PIC该怎么做，它就会忽视从PIC的下一个中断请求。从PIC连接到主PIC的第2号上，这么做OK  

下面写鼠标数据的取得方法，与键盘的完全相同，也许是因为键盘控制电路中含有鼠标控制电路，才造成了这种结果。至于传到这个设备的数据，究竟是来自键盘还是鼠标，要靠中断号码来区分  。

修改bootpack.c

```c
fifo8_init(&mousefifo, 128, mousebuf);

for(;;){
		io_cli();
		if(fifo8_status(&keyfifo) + fifo8_status(&mousefifo) == 0){
			io_stihlt();
		}
		else{
			 if(fifo8_status(&keyfifo)!=0){
				i = fifo8_get(&keyfifo);
				io_sti();
				sprintf(s, "%02X", i);
				boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
				putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
			 }
			 else if(fifo8_status(&mousefifo) != 0){
			 	i = fifo8_get(&mousefifo);
				io_sti();
				sprintf(s, "%02X", i);
				boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 32, 16, 47, 31);
				putfonts8_asc(binfo->vram, binfo->scrnx, 32, 16, COL8_FFFFFF, s);
			 }
		}
	}
```

因为鼠标往往会比键盘更快地送出大量数据，所以我们将它的FIFO缓冲区增加到了128字节  

# 4 鼠标控制

-   鼠标解读（1）

现在紧要的问题是解读这些数据，搞清楚鼠标是怎么移动的，然后结合鼠标的动作，让鼠标指针相应的动起来。

修改下主函数

```c
unsigned char mouse_dbuf[3], mouse_phase;
for(;;){
		io_cli();
		if(fifo8_status(&keyfifo) + fifo8_status(&mousefifo) == 0){
			io_stihlt();
		}
		else{
			 if(fifo8_status(&keyfifo)!=0){
				i = fifo8_get(&keyfifo);
				io_sti();
				sprintf(s, "%02X", i);
				boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
				putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
			 }
			 else if(fifo8_status(&mousefifo) != 0){
			 	i = fifo8_get(&mousefifo);
				io_sti();
				if(mouse_phase == 0){
					if(i == 0xfa){			/* 等待鼠标的0xfa的状态 */
						mouse_phase = 1;
					}
				}
				else if(mouse_phase == 1){	/* 等待鼠标的第一字节 */
					mouse_dbuf[0] = i;
					mouse_phase = 2;
				}
				else if(mouse_phase == 2){	/* 等待鼠标的第二字节 */

					mouse_dbuf[1] = i;
					mouse_phase = 3;
				}
				else if(mouse_phase == 3){	/* 等待鼠标的第三字节 */
					mouse_dbuf[2] = i;
					mouse_phase = 1;
					sprintf(s, "%02X %02X %02X", mouse_dbuf[0], mouse_dbuf[1], mouse_dbuf[2]);
					boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 32, 16, 32+8*8-1, 31);
					putfonts8_asc(binfo->vram, binfo->scrnx, 32, 16, COL8_FFFFFF, s);
				}
			 }
		}
	}
```

整理下主函数，将鼠标信息解读的部分封装成函数

```c
for(;;){
		io_cli();
		if(fifo8_status(&keyfifo) + fifo8_status(&mousefifo) == 0){
			io_stihlt();
		}
		else{
			 if(fifo8_status(&keyfifo)!=0){
				i = fifo8_get(&keyfifo);
				io_sti();
				sprintf(s, "%02X", i);
				boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 0, 16, 15, 31);
				putfonts8_asc(binfo->vram, binfo->scrnx, 0, 16, COL8_FFFFFF, s);
			 }
			 else if(fifo8_status(&mousefifo) != 0){
			 	i = fifo8_get(&mousefifo);
				io_sti();
				if(mouse_decode(&mdec, i)!=0){
					sprintf(s, "%02X %02X %02X", mdec.buf[0], mdec.buf[1], mdec.buf[2]);
					boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 32, 16, 32+8*8-1, 31);
					putfonts8_asc(binfo->vram, binfo->scrnx, 32, 16, COL8_FFFFFF, s);
				}
			 }
		}
	}
	
	
void enable_mouse(struct MOUSE_DEC *mdec){
	wait_KBC_sendready();
	io_out8(PORT_KEYCMD, KEYCMD_SENDTO_MOUSE);
	wait_KBC_sendready();
	io_out8(PORT_KEYDAT, MOUSECMD_ENABLE);
	mdec->phase = 0;
	return ;
}

int mouse_decode(struct MOUSE_DEC *mdec, unsigned char dat){
	if(mdec->phase == 0){
		if(dat == 0xfa){
			mdec->phase = 1;
		}
		return 0;
	}
	if(mdec->phase == 1){
		mdec->buf[0] = dat;
		mdec->phase = 2;
		return 0;
	}
	if(mdec->phase == 2){
		mdec->buf[1] = dat;
		mdec->phase = 3;
		return 0;
	}
	if(mdec->phase == 3){
		mdec->buf[2] = dat;
		mdec->phase = 1;
		return 1;
	}
	return -1;
}
```

-   鼠标解读（2）

修改mouse_decode函数

```c
int mouse_decode(struct MOUSE_DEC *mdec, unsigned char dat){
	if(mdec->phase == 0){
		if(dat == 0xfa){
			mdec->phase = 1;
		}
		return 0;
	}
	if(mdec->phase == 1){
		mdec->buf[0] = dat;
		mdec->phase = 2;
		return 0;
	}
	if(mdec->phase == 2){
		mdec->buf[1] = dat;
		mdec->phase = 3;
		return 0;
	}
	if(mdec->phase == 3){
		mdec->buf[2] = dat;
		mdec->phase = 1;
		mdec->btn = mdec->buf[0]&0x07;
		mdec->x = mdec->buf[1];

		mdec->y = mdec->buf[2];
		if((mdec->buf[0]&0x10)!=0){
			mdec->x |= 0xffffff00;
		}
		if((mdec->buf[0]&0x20)!=0){
			mdec->y |= 0xffffff00;
		}
		mdec->y = -mdec->y;
		return 1;
	}
	return -1;
}
```

结构体里增加的几个变量用于存放解读结果。这几个变量是x、 y和btn，分别用于存放移动
信息和鼠标按键状态  

最后的if（ mdec>phase==3）部分，是解读处理的核心。鼠标键的状态，放在buf[0]的低3位，
我们只取出这3位。十六进制的0x07相当于二进制的0000 0111，因此通过与运算（ &），可以很顺
利地取出低3位的值  

x和y，基本上是直接使用buf[1]和buf[2] ，但是需要使用第一字节中对鼠标移动有反应的几
位（参考第一节的叙述）信息，将x和y的第8位及第8位以后全部都设成1，或全部都保留为0。这
样就能正确地解读x和y。  

修改一下显示部分

```c
else if(fifo8_status(&mousefifo) != 0){
			 	i = fifo8_get(&mousefifo);
				io_sti();
				if(mouse_decode(&mdec, i)!=0){
					sprintf(s,"[lcr %4d %4d]", mdec.x, mdec.y);
					if((mdec.btn&0x01)!=0){
						s[1] = 'L';
					}
					if((mdec.btn&0x02)!=0){
						s[3] = 'R';
					}
					if((mdec.btn&0x04)!=0){
						s[2] = 'C';
					}
					boxfill8(binfo->vram, binfo->scrnx, COL8_008484, 32, 16, 32+15*8-1, 31);
					putfonts8_asc(binfo->vram, binfo->scrnx, 32, 16, COL8_FFFFFF, s);
				}
			 }
```

至此解读部分