# 1

### 简化字符串显示

当每次要打印字符串时, 程序要先涂上背景色，再在上面写字符，最后完成刷新。既然这部分重复出现，我们就把它归纳到一个函数中，这样更方便使用.

```c
void putfonts8_asc_sht(struct SHEET *sht, int x, int y, int c, int b, char *s, int l){
	boxfill8(sht->buf, sht->bxsize, b, x, y, x+l*8-1, y+15);
	putfonts8_asc(sht->buf, sht->bxsize, x, y, c, s);
	sheet_refresh(sht, x, y, x+l*8, y+16);
	return ;
}
```

这样主函数里三行就可以变一行了.

### 重新调整FIFO缓冲区

实际上我们并不需要三个缓冲区,一个就够了,这样还可以使代码精简.

# 2

### 测试性能

实际上定时器的性能并没有想像的那么好,每次都会有偏差,如果我们将count改回朴素的++,会发现每次到十秒的值是不一样的,而且在虚拟机上偏差很大.

