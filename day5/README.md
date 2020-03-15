# 1 叠加处理

今天来处理鼠标叠加和窗口叠加的问题

要在画面上叠加显示， 类似于将绘制了图案的透明图层叠加在一起。我们准备很多图层，最上面的小图层用来描绘鼠标指针。同时，我们还要通过移动图层的方法实现鼠标指针的移动以及窗口的移动。 

首先考虑将一个图层的信息编成程序

```c
struct SHEET {
	unsigned char *buf;
	int bxsize, bysize, vx0, vy0, col_inv, height, flags;
};
```

程序里的sheet这个词，表示“透明图层”的意思。笔者觉得英文里没有和“透明图层”接近的词，就凭感觉选了它。 

buf是用来记录图层上所描画内容的地址（ buffer的略语）。

图层的整体大小，用bxsize*bysize表示。 

vx0和vy0是表示图层在画面上位置的坐标， v是VRAM的略语。 col_inv表示透明色色号，它是color（颜色）和invisible（透明）的组合略语。

height表示图层高度。 Flags用于存放有关图层的各种设定信息  

显然一个图层是不够的，我们要创建管理多重图层信息的结构

```C
#define MAX_SHEETS 256
	struct SHTCTL {
		unsigned char *vram;
		int xsize, ysize, top;
		struct SHEET *sheets[MAX_SHEETS];
		struct SHEET sheets0[MAX_SHEETS];
};
```

我们创建了SHTCTL结构体，其名称来源于sheet control的略语，意思是“图层管理”。MAX_SHEETS是能够管理的最大图层数，这个值设为256应该够用了  

变量vram、 xsize、 ysize代表VRAM的地址和画面的大小，但如果每次都从BOOTINFO查询的话就太麻烦了，所以在这里预先对它们进行赋值操作。 top代表最上面图层的高度。 sheets0这个结构体用于存放我们准备的256个图层的信息。而sheets是记忆地址变量的领域，所以相应地也要先准备256份。这是干什么用呢？由于sheets0中的图层顺序混乱，所以我们把它们按照高度进行升序排列，然后将其地址写入sheets中，这样就方便多了 。 

然后创建 sheet.c

```c
struct SHTCTL *shtctl_init(struct MEMMAN *memman, unsigned char *vram, int xsize, int ysize){
	struct SHTCTL *ctl;
	int i;
	ctl = (struct SHTCTL*) memman_alloc_4k(memman, sizeof(srtuct SHTCTL));
	if(ctl == 0){
		goto err;
	}
	ctl->vram = vram;
	ctl->xsize = xsize;
	ctl->ysize = ysize;
	ctl->top = -1;		 /*一个SHEET没都有 */
	for(i=0;i<MAX_SHEETS;i++){
		ctl->sheets0[i].flags = 0;		 /* 标记为未使用 */
	}
err:
	return ctl;
}
```

首先使用memman_alloc_4k来分配用于记忆图层控制变量的内存空间，这时必须指定该变量所占空间的大小，不过我们可以使用sizeof（ struct SHTCTL）这种写法，让C编译器自动计算。只要写sizeof（变量型）， C编译器就会计算出该变量型所需的字节数  

接着，我们给控制变量赋值，给其下的所有图层变量都加上“未使用”标签。做完这一步，这个函数就完成了

下面我们再做一个函数，用于取得新生成的未使用图层

```c
struct SHEET *sheet_alloc(struct SHTCTL *ctl){
	struct SHEET *sht;
	int i;
	for(i=0; i<MAX_SHEETS; i++){
		if(ctl->sheets0[i].flags==0){
			sht = &ctl->sheets0[i];
			sht->flags = SHEET_USE;
			sht->height=-1;
			return sht;
		}
	}
	return 0;
}
```



```
void sheet_setbuf(struct SHEET *sht, unsigned char *buf, int xsize, int ysize, int col_inv){
	sht->buf = buf;
	sht->bxsize = xsize;
	sht->bysize = ysize;
	sht->col_inv = col_inv;
	return ;
}
```

这是设定图层的缓冲区大小和透明色的函数

接下来我们写设定底板高度的函数。这稍微有些复杂

```c
void sheet_updown(struct SHTCTL *ctl, struct SHEET *sht, int height){
	int h, old = sht->height;		/* 存储设置前的高度信息 */

	if(height>ctl->top + 1){	/* 如果指定的高度过高或过低，则进行修正 */
		height = ctl->top + 1;
	}
	if(height<-1){
		height = -1;
	}
	sht->height = height;		/* 设定高度 */

	/* 下面主要是进行sheets[ ]的重新排列 */
	if(old>height){			/* 比以前低 */
		if(height>=0){			/* 把中间的往上提 */
			for(h=old, h>height; h--){
				ctl->sheets[h] = ctl->sheets[h-1];
				ctl->sheets[h]->height = h;
			}
			ctl->sheets[height] = sht;
		}
		else{			/* 隐藏 */
			if(ctl->top>old){		/* 把上面的降下来 */
				for(h=old; h<ctl->top; h++){
					ctl->sheets[h] = ctl->sheets[h+1];
					ctl->sheets[h]->height = h;
				}
			}
			ctl->top--;		/* 由于显示中的图层减少了一个，所以最上面的图层高度下降 */
		}
		sheet_refresh(ctl);		/* 按新图层的信息重新绘制画面 */
	}
	else if(old<height){		/* 比以前高 */
		if(old>=0){
			for(h=old; h<height; h++){	/* 把中间的拉下去 */
				ctl->sheets[h] = ctl->sheets[h+1];
				ctl->sheets[h]->height = h;
			}
			ctl->sheets[height] = sht;
		}
		else{		/* 由隐藏状态转为显示状态 */
			for(h=ctl->top; h>=height; h--){	/* 将已在上面的提上来 */
				ctl->sheets[h+1] = ctl->sheets[h];
				ctl->sheets[h+1]->height = h+1;
			}
			ctl->sheets[height] = sht;
			ctl->top++;			/* 由于已显示的图层增加了1个，所以最上面的图层高度增加 */
		}
	sheet_refresh(ctl);			/* 按新图层信息重新绘制画面 */
	}
	return ;
}
```

下面来说说在sheet_updown中使用的sheet_refresh函数

```c
void sheet_refresh(struct SHTCTL *ctl){
	int h, bx, by, vx, vy;
	unsigned char *buf, c, *vram = ctl->vram;
	struct SHEET *sht;
	for(h=0; h<= ctl->top; h++){
		sht=ctl->sheets[h];
		buf = sht->buf;
		for(by=0; by<sht->bysize; by++){
			vy = sht->vy0+by;
			for(bx=0;bx<sht->bxsize; bx++){
				vx = sht->vx0+bx;
				c = buf[by*sht->bxsize+bx];
				if(c!=sht->col_inv){
					vram[vy*ct->xsize+vx] = c;
				}
			}
		}
	}
	return ;
}
```

对于已设定了高度的所有图层而言，要从下往上，将透明以外的所有像素都复制到VRAM中。由于是从下开始复制，所以最后最上面的内容就留在了画面上

现在我们来看一下不改变图层高度而只上下左右移动图层的函数——sheet_slide。slide原意 是“滑动”，这里指上下左右移动图层。

```
void sheet_slide(struct SHTCTL *ctl, struct SHEET *sht, int vx0, int vy0) {
	sht->vx0 = vx0; sht->vy0 = vy0;
	if (sht->height >= 0) { /* 如果正在显示*/ 
		sheet_refresh(ctl); /* 按新图层的信息刷新画面 */
	} 
	return;
}
```

最后是释放已使用图层的内存的函数sheet_free

```c
void sheet_free(struct SHTCTL *ctl, struct SHEET *sht){
	if(sht->height>=0){
		sheet_updown(ctl, sht, -1);/* 如果处于显示状态，则先设定为隐藏 */
	}
	sht->flags = 0;/* "未使用"标志 */
	return ;
}
```

然后就要改造主函数了

详见主函数

### 提高画面叠加处理的速度

鼠标指针虽然最多只有16×16=256个像素，可根据harib07b的原理，只要它稍一移动，程序
就会对整个画面进行刷新，也就是重新描绘320×200=64 000个像素。而实际上，只重新描绘移动 相关的部分，也就是移动前后的部分就可以了，即256×2=512个像素。这只是64 000像素的0.8% 而已，所以有望提速很多。现在我们根据这个思路写一下程序

```c

void sheet_refreshsub(struct SHTCTL *ctl, int vx0, int vy0, int vx1, int vy1){
	int h, bx, by, vx, vy;
	unsigned char *buf, c, *vram = ctl->vram;
	struct SHEET *sht;
	for(h=0; h<=ctl->top; h++){
		sht = ctl->sheets[h];
		buf = sht->buf;
		for(by = 0; by<sht->bysize; by++){
			vy = sht->vy0+by;
			for(bx=0; bx<sht->bxsize; bx++){
				vx = sht->vx0+bx;
				if(vx0<=vx&&vx<vx1&&vy0<=vy&&vy<vy1){
					c = buf[by*sht->bxsize+bx];
					if(c!=sht->col_inv){
						vram[vy*ctl->xsize+vx] = c;
					}
				}
			}
		}
	}
	return ;
}
```

然后改写sheet_slide

```c
void sheet_slide(struct SHTCTL *ctl, struct SHEET *sht, int vx0, int vy0){
	int old_vx0 = sht->vx0, old_vy0 = sht->vy0;
	sht->vx0 = vx0;
	sht->vy0 = vy0;
	if(sht->height>=0){	/* 如果正在显示，则按新图层的信息刷新画面 */
		sheet_refreshsub(ctl, old_vx0, old_vy0, old_vx0 + sht->bxsize, old_vy0 + sht->bysize);
		sheet_refreshsub(ctl, vx0, vy0, vx0 + sht->bxsize, vy0 + sht->bysize);
	}
	return ;
}

```

估计大家会认为“这次鼠标的移动就快了吧”，但移动鼠标时，由于要在画面上显示坐标等信息，结果又执行了sheet_refresh程序，所以还是很慢。为了不浪费我们付出的各种努力，下面 我们就来解决一下图层内文字显示的问题

我们所说的在图层上显示文字，实际上并不是改写图层的全部内容。假设我们已经写了20个字，那么8×16×20=2560，也就是仅仅重写2560个像素的内容就应该足够了。但现在每次却要 重写64 000个像素的内容，所以速度才那么慢

这么说来，这里好像也可以使用refreshsub，那么我们就来重新编写函数sheet_refresh吧

所谓指定范围，并不是直接指定画面内的坐标，而是以缓冲区内的坐标来表示。这样一来， HariMain就可以不考虑图层在画面中的位置了。

我们改动了refresh，所以也要相应改造updown。做了改动的只有sheet_refresh（ctl）这部分
（有两处），修改后的程序如下

```c
sheet_refreshsub(ctl, sht->vx0, sht->vy0, sht->vx0 + sht->bxsize, sht->vy0 + sht->bysize);
```

最后还要改写HariMain。

这里我们仅仅改写了sheet_refresh，变更点共有4个。只有每次要往buf_back中写入信息时， 才进行sheet_refresh。

-   再次提高叠加处理速度

然而refreshsub还是不够快

即使不写入像素内容，也要多次执行if语句，这一点不太好，如果能改善一 下，速度应该会提高不少

按照上面这种写法，即便只刷新图层的一部分，也要对所有图层的全部像素执行if语句，判断“是写入呢，还是不写呢”。而对于刷新范围以外的部分，就算执行if判断语句，最后也不会进 行刷新，所以这纯粹就是一种浪费。既然如此，我们最初就应该把for语句的范围限定在刷新范围 之内

```c
void sheet_refreshsub(struct SHTCTL *ctl, int vx0, int vy0, int vx1, int vy1){
	int h, bx, by, vx, vy, bx0, by0, bx1, by1;
	unsigned char *buf, c, *vram = ctl->vram;
	struct SHEET *sht;
	for(h=0; h<=ctl->top; h++){
		sht = ctl->sheets[h];
		buf = sht->buf;
		/* 使用vx0～vy1，对bx0～by1进行倒推 */
		bx0 = vx0 - sht->vx0;
		by0 = vy0 - sht->vy0;
		bx1 = vx1 - sht->vx0;
		by1 = vy1 - sht->vy0;
		if(bx0<0){ bx0 = 0; }	/* 说明(1) */
		if(by0<0){ by0 = 0; }
		if(bx1>sht->bxsize){ bx1 = sht->bxsize; }		/* 说明(2) */
		if(by1>sht->bysize){ by1 = sht->bysize; }
		for(by = by0; by<by1; by++){
			vy = sht->vy0+by;
			for(bx = bx0; bx<bx1; bx++){
				vx = sht->vx0+bx;
				c = buf[by*sht->bxsize+bx];
				if(c!=sht->col_inv){
					vram[vy*ctl->xsize+vx] = c;
				}
			}
		}
	}
	return ;
}
```

改良的关键在于，bx在for语句中并不是在0到bxsize之间循环，而是在bx0到bx1之间循环（对于by也一样）。而bx0和bx1都是从刷新范围“倒推”求得的。倒推其实就是把公式变形转换了一 下，具体如下：
vx = sht->vx0 + bx; → bx = vx - sht->vx0;

计算vx0的坐标相当于bx中的哪个位置，然后把它作为bx0。其他的坐标处理方法也一样。

这样算完以后，就该执行以上程序中说明(1)的地方了。这行代码用于处理刷新范围在图层外
侧的情况。什么时候会出现这种情况呢？比如在sht_back中写入字符并进行刷新，而且刷新范围 的一部分被鼠标覆盖的情况

程序中“说明(2)”部分所做的，是为了应对不同的重叠方式。

第三种情况是完全不重叠的情况。例如，鼠标的图层往左移动直至不再重叠。此时当然完全不需要进行重复描绘，那么程序是否可以正常运行呢？ 利用倒推计算得出的bx0和bx1都是负值，在说明(1中，仅仅bx0被修正为0，而在说明(2) 中bx1没有被修正，还是负的。这样的话，for（bx = bx0;bx < bx1;bx++）这个语句里的循环条件bx < bx1 从最开就不成立，所以for语句中的命令得不到循环，这样就完全不会进行重复描绘了，很好

## 2 鼠标显示的问题（边缘显示） ctl的指定省略 窗口

在Windows中，鼠标应该可以向右或向下移动到画面之外隐藏起来的，可是我们的操作系统却还不能实现这 样的功能，这多少有些遗憾

我们小小的修改一下主函数防止鼠标出画面的代码

然后发现一出外面右边就会渲染一个鼠标出来，只要图层一跑到画面的外面去就会出问题

怎么才能让图层位于画面以外时也不出问题呢？因为只有sheet_refreshsub函数在做把图层
2 内容写入VRAM的工作，所以我们要把这个函数做得完美一些，让它不刷新画面以外的部分

-   进行ctl的指定省略

其实笔者对sheet_updown函数不太满意，因为仅是上下移动图层，就必须指定ctl，太麻烦

要想改善这个问题，首先我们需要在struct SHEET中加入struct SHTCTL *ctl 。

然后对函数shtctl_init也进行追加，仅追加1行即可。

然后剩余函数删除ctl形参

-   窗口

其实方法很简单，就像前面制作背景和鼠标那样，只要先准备一张图层，然后在图层缓冲
区内描绘一个貌似窗口的图就可以了。那么我们就来制作一个具有这种功能的函数 make_window8

见主程序