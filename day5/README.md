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

