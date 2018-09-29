## GDAL + Visual Studio 配置 ##

### GDAL相关知识 ###

1. Class    `GDALDataset` 	*A set of associated raster bands, usually from one file*

   - Created with `GDALOpenShared("file_path",GA_ReadOnly)` for shared dataset   

     ~~~c++
     ```poSrcDS = (GDALDataset*)GDALOpenShared(srcPath, GA_ReadOnly);```
     ~~~

   - `GDALClose(GDALDataset)` to destory the GDALDataset  

     ```c++
     GDALClose(poSrcDS);
     ```

   - `GetRasterXSize() GetRasterYSize() GetRasterCount()`  Fetch raster width in pixels, fetch raster height  in pixels and fetch raster bands on this dataset
   - `GetRasterBand(int)` Fetch a band object `GDALRasterBand `   

2.  Class `GetGDALDriverManager`  *Class for managing the registration of file format drivers*

   - `GetDriverByName(const char *)` Fetch a driver based on the short name `GDALDriver`

3.  Class `GDALDriver` *Format specific driver*

   - `Create (const char *pszName, int nXSize, int nYSize, int nBands, GDALDataType eType, char **papszOptions)` Create a new dataset with this driver

     ```c++ 
     poDstDS = GetGDALDriverManager()->GetDriverByName("GTiff")->Create(dstPath, imgXlen, imgYlen, bandNum, GDT_Byte, NULL); 
     ```

4. `CPLMalloc()` and `CPLFree()` 

   ```c++
   bufftmp = (GByte*)CPLMalloc(imgXlen*imgYlen * sizeof(GByte));
   CPLFree(bufftmp);
   ```

5.  Class     `GDALRasterBand`   *A single raster band (or channel)*

   - `RasterIO(GDALRWFlag *eRWFlag*,int *nXOff*,int *nYOff*,int *nXSize*,int *nYSize*,void * *pData*,int *nBufXSize*,int *nBufYSize*,GDALDataType *eBufType*,GSpacing *nPixelSpace*,GSpacing *nLineSpace*,GDALRasterIOExtraArg * *psExtraArg* )`  Read raster band data from GDALRasterBand to pData or write raster band data from pData to GDALRasterBand

     ``````c++
     poSrcDS->GetRasterBand(i + 1)->RasterIO(GF_Read, 0, 0, imgXlen, imgYlen, bufftmp, imgXlen, imgYlen, GDT_Byte, 0, 0);
     		poDstDS->GetRasterBand(i + 1)->RasterIO(GF_Write, 0, 0, imgXlen, imgYlen, bufftmp, imgXlen, imgYlen, GDT_Byte, 0, 0);
     ``````

### 主要步骤 ###

1. 使用VS2017新建Win32控制台应用程序GDALDemo

2. 将头文件、lib文件放在项目内，并加入预编译代码`#include "gdal/gdal_priv.h"
   #pragma comment(lib, "gdal_i.lib")`  

3. 按照文档进行代码编写，主要类和方法在基本知识已经介绍

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
     	char* dstPath = "result.tif";
     	GByte* bufftmp;
     	int bandNum;
     
     	GDALAllRegister();
     
     	poSrcDS = (GDALDataset*)GDALOpenShared(srcPath, GA_ReadOnly);
     
     	imgXlen = poSrcDS->GetRasterXSize();
     	imgYlen = poSrcDS->GetRasterYSize();
     	bandNum = poSrcDS->GetRasterCount();
     
     	cout << "Image X Length:" << imgXlen << endl;
     	cout << "Image Y Length:" << imgYlen << endl;
     	cout << "Band Number:" << bandNum << endl;
     
     	bufftmp = (GByte*)CPLMalloc(imgXlen*imgYlen * sizeof(GByte));
     
     	poDstDS = GetGDALDriverManager()->GetDriverByName("GTiff")->Create(dstPath, imgXlen, imgYlen, bandNum, GDT_Byte, NULL);
     
     	for (int i = 0; i < bandNum; i++)
     	{
     		poSrcDS->GetRasterBand(i + 1)->RasterIO(GF_Read, 0, 0, imgXlen, imgYlen, bufftmp, imgXlen, imgYlen, GDT_Byte, 0, 0);
     		poDstDS->GetRasterBand(i + 1)->RasterIO(GF_Write, 0, 0, imgXlen, imgYlen, bufftmp, imgXlen, imgYlen, GDT_Byte, 0, 0);
     	}
     	CPLFree(bufftmp);
     
     	GDALClose(poSrcDS);
     	GDALClose(poDstDS);
     
     	system("pause");
     	return 0;
     }
     ```

4. 最后直接生成Release版本

5. 在Release文件夹内放入dll动态链接库文件，用于运行时加载，把所需源图片也加进去

6. 运行exe，输出结果如下，并生成tif文件![](http://ww1.sinaimg.cn/large/006cY7n0ly1fvqy71yivpj30kc07a74e.jpg)

### 遇到问题 ###

1. 首先有实例代码，所以减轻了很大难度，但是还是有很多实例代码只是涉略一点，于是花了很多时间看了一波官方的文档
2. 在项目生成的时候，一开始选择了x64，发现编译失败，有点迷。但是换了x86就OK了，应该是lib库是32bit的。