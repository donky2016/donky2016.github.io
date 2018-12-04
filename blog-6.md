## 遥感图像的分块处理 ##
-----
### 基础原理 ###
- 在遥感图像处理中一般采取分块读写的方法，因为遥感图像尺寸巨大（长宽可达30000像素以上），不可能开辟一个大内存把整个图像读进来。分块读取的道理一般大家都懂，但处理上却是有学问的。

- 在大图像处理中磁盘I/O一般是效率的主要瓶颈。因此如何分块的着眼点应该是如何减少磁盘I/O。一般的图像处理系统采取将块分成256 * 256或者512 * 512的块。实际上这是不利于减少磁盘I/O次数。

- 一般而言图像文件在磁盘上是按行存贮的，就是从第一行到最后一行的，那么我们很快就可以看到其中弊端，就是当将数据写入到某一块时，其写入顺序是怎样的呢？首先当然是从块的起始地址写，将块的第一行的数据写入，这时你要问块的第二行数据和第一行数据在连续的？很显然，它们不是连续的。那么你读写时必须先移动文件指针，那么读取一块数据你就需要移动256次文件指针。整幅图像就需要至少移动块数 * 256次文件指针，这样的磁盘I/O次数有点惊人。

- 这种分块方法是采取一次读取若干行的方法。这种方法有两个好处：首先降低了程序的逻辑复杂度，每块只是行号变化，块的起始列位置都是0，不需要变化；另一方面，这种分块能大大减少磁盘的I/O次数，这是因为每一块的每行数据在磁盘上都是连续的，因此在读写时只需将文件指针定位到块的起始位置，就能实现整块的读写。毫无疑问，这种分块方法比前一种分块方法磁盘I/O次数要少很多。  

### 实验代码 ###
GDALDemo.cpp
```cpp
void IHSBigImageFusionByBlock(char *srcMULPath, char *srcPANPath, char * resPath)
{
	/*
		分块图像融合
		param: 多光谱图像路径，全色图像路径，融合结果图像路径
	*/
	GDALAllRegister();
	// 块的大小
	int blockLen = 256;
	GDALDataset* srcMULDS = (GDALDataset*)GDALOpenShared(srcMULPath, GA_ReadOnly);
	GDALDataset* srcPANDS = (GDALDataset*)GDALOpenShared(srcPANPath, GA_ReadOnly);

	int imgMULXlen = srcMULDS->GetRasterXSize();
	int imgMULYlen = srcMULDS->GetRasterYSize();
	int MULBandNum = srcMULDS->GetRasterCount();

	GDALDataset *resDS = GetGDALDriverManager()->GetDriverByName("GTiff")->Create(resPath, imgMULXlen, imgMULYlen, MULBandNum, GDT_Byte, NULL);

	cout << "MUL image X len: " << imgMULXlen << endl;
	cout << "MUL image Y len: " << imgMULYlen << endl;
	cout << "MUL image Band num : " << MULBandNum << endl;

	int imgPANXlen = srcPANDS->GetRasterXSize();
	int imgPANYlen = srcPANDS->GetRasterYSize();
	int PANBandNum = srcPANDS->GetRasterCount();
	cout << "PAN image X len: " << imgPANXlen << endl;
	cout << "PAN image Y len: " << imgPANYlen << endl;
	cout << "PAN image Band num : " << PANBandNum << endl;

	// 计算出右边剩余的像素和下面剩余的像素
	int blocksX = imgMULXlen / blockLen;
	int blocksY = imgMULYlen / blockLen;
	int restX = imgMULXlen - blocksX * blockLen;
	int restY = imgMULYlen - blocksY * blockLen;
	cout << "Blocks num: X-" << blocksX << "   Y-" << blocksY << endl;
	cout << "Rest pixels: X-" << restX << "   Y-" << restY << endl;

	float *panBuffTmp;
	float **RGB;
	float **resultRGB;
	float **IHS;
	RGB = (float**)CPLMalloc(MULBandNum * sizeof(float));
	resultRGB = (float**)CPLMalloc(MULBandNum * sizeof(float));
	IHS = (float**)CPLMalloc(MULBandNum * sizeof(float));
	panBuffTmp = (float*)CPLMalloc(blockLen*blockLen * sizeof(float));

	for (int k = 0; k < 3; k++)
	{
		RGB[k] = (float*)CPLMalloc(blockLen*blockLen * sizeof(float));
		IHS[k] = (float*)CPLMalloc(blockLen*blockLen * sizeof(float));
		resultRGB[k] = (float*)CPLMalloc(blockLen*blockLen * sizeof(float));
	}
	int X,Y;
	for (int m = 0; m <= blocksY; m++)
	{
		for (int n = 0; n <= blocksX; n++)
		{
			// 计算块的实际大小
			if (n == blocksX)
			{
				X = restX;
			}
			else if (m == blocksY)
			{
				Y = restY;
			}
			else
			{
				X = Y = blockLen;
			}
			//cout << "Working..." << endl;
			//cout << "Xoff:" << n*blockLen << "    Yoff:" << m*blockLen << endl;
			//cout << "Block size:" << X << "*" << Y << "=" << X*Y << endl;
			
			// 根据块的大小来从图像中读取数据
			srcPANDS->GetRasterBand(1)->RasterIO(GF_Read, n*blockLen, m*blockLen, X, Y, panBuffTmp, X, Y, GDT_Float32, 0, 0);
			for (int j = 0; j < 3; j++)
			{		
				srcMULDS->GetRasterBand(j + 1)->RasterIO(GF_Read, n*blockLen, m*blockLen, X, Y, RGB[j], X, Y, GDT_Float32, 0, 0);
			}

			// 核心算法融合
			IHSImageFusion(RGB, panBuffTmp, IHS, resultRGB, X, Y);

			// 数据写入图像
			for (int i = 0; i < 3; i++)  
			{
				resDS->GetRasterBand(i + 1)->RasterIO(GF_Write, n*blockLen, m*blockLen, X, Y, resultRGB[i], X, Y, GDT_Float32, 0, 0);
			}
		}
	}
	// 释放内存
	CPLFree(panBuffTmp);
	CPLFree(RGB);
	CPLFree(IHS);
	CPLFree(resultRGB);
	GDALClose(srcMULDS);
	GDALClose(srcPANDS);
	GDALClose(resDS);
}
void IHSBigImageFusionByLine(char *srcMULPath, char *srcPANPath, char * resPath)
{
	/*
		分行图像融合
		param: 多光谱图像路径，全色图像路径，融合结果图像路径
	*/
	GDALAllRegister();

	GDALDataset* srcMULDS = (GDALDataset*)GDALOpenShared(srcMULPath, GA_ReadOnly);
	GDALDataset* srcPANDS = (GDALDataset*)GDALOpenShared(srcPANPath, GA_ReadOnly);

	int imgMULXlen = srcMULDS->GetRasterXSize();
	int imgMULYlen = srcMULDS->GetRasterYSize();
	int MULBandNum = srcMULDS->GetRasterCount();
	int lineHeight = 256;		// 行高度
	int lineLen = imgMULXlen;	// 行长度

	GDALDataset *resDS = GetGDALDriverManager()->GetDriverByName("GTiff")->Create(resPath, imgMULXlen, imgMULYlen, MULBandNum, GDT_Byte, NULL);

	cout << "MUL image X len: " << imgMULXlen << endl;
	cout << "MUL image Y len: " << imgMULYlen << endl;
	cout << "MUL image Band num : " << MULBandNum << endl;

	int imgPANXlen = srcPANDS->GetRasterXSize();
	int imgPANYlen = srcPANDS->GetRasterYSize();
	int PANBandNum = srcPANDS->GetRasterCount();
	cout << "PAN image X len: " << imgPANXlen << endl;
	cout << "PAN image Y len: " << imgPANYlen << endl;
	cout << "PAN image Band num : " << PANBandNum << endl;

	// 计算剩余最后不足规定行高度的行的高度
	int linesY = imgMULYlen / lineHeight;
	int restY = imgMULYlen - lineHeight * linesY;
	cout << "Line num:" << linesY << endl;
	cout << "Rest pixel:" << restY << endl;

	float *panBuffTmp;
	float **RGB;
	float **resultRGB;
	float **IHS;
	RGB = (float**)CPLMalloc(MULBandNum * sizeof(float));
	resultRGB = (float**)CPLMalloc(MULBandNum * sizeof(float));
	IHS = (float**)CPLMalloc(MULBandNum * sizeof(float));
	panBuffTmp = (float*)CPLMalloc(lineHeight*lineLen * sizeof(float));

	for (int k = 0; k < 3; k++)
	{
		RGB[k] = (float*)CPLMalloc(lineHeight*lineLen * sizeof(float));
		IHS[k] = (float*)CPLMalloc(lineHeight*lineLen * sizeof(float));
		resultRGB[k] = (float*)CPLMalloc(lineHeight*lineLen * sizeof(float));
	}
	int Y;
	for (int i = 0; i <= linesY; i++)
	{
		// 计算行的实际高度
		if (i == linesY)
		{
			Y = restY;
		}
		else
		{
			Y = lineHeight;
		}
		cout << "Working..." << endl;
		cout << "Line " << i << "   Xoff:0  Yoff:" << i*lineHeight << endl;
		cout << "Line size:" << Y << "*" << lineLen << "=" << lineLen*Y << endl;
		// 读取图像数据
		srcPANDS->GetRasterBand(1)->RasterIO(GF_Read, 0, i*lineHeight, lineLen, Y, panBuffTmp, lineLen, Y, GDT_Float32, 0, 0);
		for (int j = 0; j < 3; j++)
		{
			srcMULDS->GetRasterBand(j + 1)->RasterIO(GF_Read, 0, i*lineHeight, lineLen, Y, RGB[j], lineLen, Y, GDT_Float32, 0, 0);
		}
		// 核心算法计算
		IHSImageFusion(RGB, panBuffTmp, IHS, resultRGB, lineLen, Y);
		// 数据写入图像
		for (int k = 0; k < 3; k++)
		{
			resDS->GetRasterBand(k + 1)->RasterIO(GF_Write, 0, i*lineHeight, lineLen, Y, resultRGB[k], lineLen, Y, GDT_Float32, 0, 0);
		}
	}
	// 释放内存
	CPLFree(panBuffTmp);
	CPLFree(RGB);
	CPLFree(IHS);
	CPLFree(resultRGB);
	GDALClose(srcMULDS);
	GDALClose(srcPANDS);
	GDALClose(resDS);
}

int main()
{
	char* srcMULPath = "Data\\pictures\\Mul_large.tif";
	char* srcPANPath = "Data\\pictures\\Pan_large.tif";
	char* resPathByBlock= "Data\\output\\Big-image-fusion-block.tif";
	char* resPathByLine = "Data\\output\\Big-image-fusion-line.tif";
    // 记录时间
	int t1 = clock();
	IHSBigImageFusionByLine(srcMULPath, srcPANPath, resPathByLine);
	int t2 = clock();
	IHSBigImageFusionByBlock(srcMULPath, srcPANPath, resPathByBlock);
	int t3 = clock();
    // 输出时间
	cout << "Image fusion by line time:" << (t2 - t1) / CLOCKS_PER_SEC << endl;
	cout << "Image fusion by block time:" << (t3 - t2) / CLOCKS_PER_SEC << endl;
	system("pause");
	return 0;
}
```
### 实验结果 ###
   - 图片太大，下载地址[http://donky.top/img/bed/Big-image-fusion.tif](http://donky.top/img/bed/Big-image-fusion.tif)
   - 所用时间
    ![](http://donky.top/img/bed/compare-time.jpg)

### 实验心得 ###
- 抽象出函数的时候需要初始化一些变量，不然会导致一些连续操作如连加，会使程序结果错误
- 在需要使用大量内存的时候，在算法函数外进行申请内存，若在算法函数内，会一直申请内存，若在算法函数内进行内存的释放，会增加内存的申请与释放时间
- 算法函数内需要进行赋值操作时，应该使用赋值操作，不应该使用指针赋值，这样会使两个指针指向相同的地址，导致下一次调用函数时发生错误