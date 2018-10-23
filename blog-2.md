## GDAL修改图像block数据 ##
-----
### 相关知识 ###
- 数字图像是指物理图像的连续信号值被离散化后，由被称作像素的小块区域组成的二维矩阵。

- 将物理图像行列划分后，每个小块区域称为像素（Pixel）。每个像素包括两个属性：位置和色彩（或亮度）。

- 灰度图像，每个像素的亮度用一个整数来表示，通常数值范围在0到255之间，0表示纯黑、255表示纯白，而其它表示灰色。在计算机中存储时，使用格式为 unsigned char 。GDAL中为 GByte (头文件中有这样的定义：typedef unsigned char GByte)
- 彩色图像可以用“红、绿、蓝”三基色的二维矩阵来表示。通常，三基色的每个数值也是在0到255之间，0表示相应的基色在该像素中没有，而255则代表相应的基色在该像素中取得最大值。
- GDAL相关知识
  - GetRasterBand(1)得到的是红色通道；
  - GetRasterBand(2)得到的是绿色通道；
  - GetRasterBand(3)得到的是蓝色通道；

### 编程实现 ###

- GDALDemo.cpp

  ```c++
  // GDALDemo.cpp : 定义控制台应用程序的入口点。
  //
  
  #include "stdafx.h"
  #include "gdal/gdal_priv.h"
  #include<iostream>
  #pragma comment(lib, "gdal_i.lib")
  
  using namespace std;
  int main()
  {
  	GDALDataset* poSrcDS;
  	GDALDataset* poDstDS;
  	int imgXlen, imgYlen;
  	char* srcPath = "Taeyeon.png";
  	char* dstPath = "result_modify.tif";
  	GByte* bufftmp;
  	int bandNum;
  	// 修改图像block数据
  	int blackStartX = 300, blackStartY = 300;
  	int whiteStartX = 500, whiteStartY = 500;
  	int blackXlen = 100;
  	int blackYlen = 50;
  	int whiteXlen = 50;
  	int whiteYlen = 100;
  
  	GDALAllRegister();
  
  	poSrcDS = (GDALDataset*)GDALOpenShared(srcPath, GA_ReadOnly);
  
  	imgXlen = poSrcDS->GetRasterXSize();
  	imgYlen = poSrcDS->GetRasterYSize();
  	bandNum = poSrcDS->GetRasterCount();
  
  	cout << "Image X Length:" << imgXlen << endl;
  	cout << "Image Y Length:" << imgYlen << endl;
  	cout << "Band Number:" << bandNum << endl;
  	// 分配内存
  	bufftmp = (GByte*)CPLMalloc(imgXlen*imgYlen * sizeof(GByte));
  
  	// 加载驱动
  	poDstDS = GetGDALDriverManager()->GetDriverByName("GTiff")->Create(dstPath, imgXlen, imgYlen, bandNum, GDT_Byte, NULL);
  
  	for (int i = 0; i < bandNum; i++)
  	{
  		// 读取图像数据到内存
  		poSrcDS->GetRasterBand(i + 1)->RasterIO(GF_Read, 0, 0, imgXlen, imgYlen, bufftmp, imgXlen, imgYlen, GDT_Byte, 0, 0);		
  		// 将内存中数据写入图像
  		poDstDS->GetRasterBand(i + 1)->RasterIO(GF_Write, 0, 0, imgXlen, imgYlen, bufftmp, imgXlen, imgYlen, GDT_Byte, 0, 0);
  		// 修改内存中部分数据
  		for (int k = 0; k < blackYlen; k++)
  		{
  			for (int j = 0; j < blackXlen; j++)
  			{
  				bufftmp[k*blackXlen + j] = (GByte)255;
  			}
  		}
  		poDstDS->GetRasterBand(i + 1)->RasterIO(GF_Write, blackStartX, blackStartY, blackXlen, blackYlen, bufftmp, blackXlen, blackYlen, GDT_Byte, 0, 0);
  		for (int n = 0; n < whiteYlen; n++)
  		{
  			for (int m = 0; m < whiteXlen; m++)
  			{
  				bufftmp[n*whiteXlen + m ] = (GByte)0;
  
  			}
  		}
  		poDstDS->GetRasterBand(i + 1)->RasterIO(GF_Write, whiteStartX, whiteStartY, whiteXlen, whiteYlen, bufftmp, whiteXlen, whiteYlen, GDT_Byte, 0, 0);
  	}
  	CPLFree(bufftmp);
  
  	GDALClose(poSrcDS);
  	GDALClose(poDstDS);
  
  	system("pause");
  	return 0;
  }
  ```

- 结果

  ![](http://ww1.sinaimg.cn/large/006cY7n0ly1fw0pta5olej312w0j1abt.jpg)

### 问题与体会 ###

1. 在第一个程序上加以修改，本来想实现一次修改bufftmp，从而可以一次将数据输入到图像里，但是发现其中的bufftmp数据，很难正确修改矩阵中的一个block，放弃了。
2. 发现RasterIO方法的Write，可以在某一固定位置上直接添加一个block。