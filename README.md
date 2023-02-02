# JPEG Decoder

### 基于ujpeg的能够解析大话西游2旧地图的解码器

### 引用
如果想要学习jpeg压缩与解析原理，推荐阅读[跟我写 JPEG 解码器](https://github.com/MROS/jpeg_tutorial)和云风的[JPEG简易文档](https://www.codingnow.com/2000/download/jpeg.txt)

---
#### JPEG 规范的是一组而非一个算法
JPEG 这个标准其实定义了多种压缩算法以及编码方式
我们先以压缩算法来加以区分，共有四种：

- 循序（sequential）
- 递进（progressive）
- 层次（hierarchical）
- 无损（lossless）

sequential 会由上而下解码
![sequential](https://raw.githubusercontent.com/MROS/jpeg_tutorial/master/doc/image/sequential.gif )

progressive 则在解码的过程中，从模糊渐渐变得清晰
![progressive ](https://raw.githubusercontent.com/MROS/jpeg_tutorial/master/doc/image/progressive.gif )
---
#### baseline
所有 JPEG 规范组合中，最为常见和简单的一种就是 baseline ，它的基本特征如下

- 循序（sequential）
- 霍夫曼编码
- 精度为 8bit

这份代码也仅限于解码 baseline  JPEG 。


### JPEG 编/解码流程
---
先看一张最简单的图
![](https://raw.githubusercontent.com/MROS/jpeg_tutorial/master/doc/image/%E7%B7%A8%E8%A7%A3%E7%A2%BC%E7%95%A5%E5%9C%96.jpg)

相信这张图非常简单易懂，我们接着将压缩与解压缩的流程放大来看，baseline JPEG 会透过 DCT 变换，将原色彩空间映射到新空间，再使用霍夫曼编码尝试进行压缩。

解码时则全部反过来，如下示意图
![](https://raw.githubusercontent.com/MROS/jpeg_tutorial/master/doc/image/%E7%B7%A8%E8%A7%A3%E7%A2%BC%E7%95%A5%E5%9C%962.jpg)

DCT 变换以及霍夫曼编码，都是完全可逆的运算，也就是说这样得到的压缩文件是无损的，为了进一步提高压缩率， baseline JPEG 还会抛弃掉一些人类视觉上感受不明显的信息：

*   将颜色从RGB空间转换成YCbCr空间后，降采样
*   在DCT变换之后，量化新空间里的数据

以上这两点都是有损的，是降低压缩数据大小的最主要流程，加入这两项之后，我们将示意图更新为
![](https://raw.githubusercontent.com/MROS/jpeg_tutorial/master/doc/image/%E7%B7%A8%E8%A7%A3%E7%A2%BC%E7%95%A5%E5%9C%963.jpg)

这张图差不多把 JPEG 编解码的主要流程画出来了，但要写一个解码器，这些知识是远远不够的，我们仍得了解更多细节。

### 大话2旧地图的区别
--- 

在大话西游2旧地图中，在每行MCU前，额外插入了7字节数据，分别是这一行的YCbCr的DC值和1字节的字节bit坐标偏移量，因此需要我们在处理每一行的MCU开始前，先将该DC值读取出来，然后调整bit坐标偏移量


在普通的JPEG中，除第一个外，每个MCU都继承上个MCU单元的处理后的DC值，而在旧地图中，新的一行MCU的初始DC值，被提取到了这一行MCU数据之前，因此在普通JPEG DECODER看来本该是下一个MCU数据的起始位置的地方，多了7字节的数据（包含6字节DC值和一字节bit坐标），也因此导致普通的JPEG DECODER无法解析旧地图的JPEG。

以下是云风版jpeg decoder用于解析大话西游2旧地图的关键代码

```
for (i=0;i<VDU;i+=16) {
	jpeg_stream+=(jpeg_bit+7)/8;
	jpeg_DC[0]=*jpeg_stream_short++;  // 关键
	jpeg_DC[1]=*jpeg_stream_short++;  // 关键
	jpeg_DC[2]=*jpeg_stream_short++;  // 关键
	jpeg_bit=*jpeg_stream++;          // 关键  一字节bit坐标

	for (j=0;j<HDU;j+=16) {
		if (*(jpeg_stream+(jpeg_bit+7)/8)==0xff) {
			jpeg_stream+=(jpeg_bit+7)/8;
			jpeg_bit=0;
			while (*jpeg_stream!=0xff) ++jpeg_stream;
				++jpeg_stream;
				jpeg_DC[0]=jpeg_DC[1]=jpeg_DC[2]=0;
		}
		jpeg_decode_DU(jpeg_ybuf,0);
		jpeg_decode_DU(jpeg_ybuf+64,0);
		jpeg_decode_DU(jpeg_ybuf+128,0);
		jpeg_decode_DU(jpeg_ybuf+192,0);
		jpeg_decode_DU(jpeg_cbbuf,1);
		jpeg_decode_DU(jpeg_crbuf,2);
		YCbCr411((unsigned short*)(output_buffer+i*pitch)+j,pitch);
	}
}
```


而在ujpeg版中，处理也类似，但麻烦的一点是原数据的bit坐标偏移是基于每次读取32bit的，而ujpeg每次读取8bit，需要额外处理

```c++
// 大话2旧地图特殊处理
if (uj->mapx) {
  int back = uj->bufbits / 8;
  while (back) {
    uj->pos--;
    uj->size++;
    back--;  
  }
  uj->buf = 0;
  uj->bufbits = 0;
  uj->comp[0].dcpred = (short)ujDecode16Reverse(uj->pos);
  uj->pos += 2;
  uj->comp[1].dcpred = (short)ujDecode16Reverse(uj->pos);
  uj->pos += 2;
  uj->comp[2].dcpred = (short)ujDecode16Reverse(uj->pos);
  uj->pos += 2;
  int bit_pos = *uj->pos++;
  uj->size -= 7;
  ujSkipBits(uj, bit_pos);
}
```
