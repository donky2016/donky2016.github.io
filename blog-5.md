## IHS遥感图像融合 ##
-----
### 基础知识 ###
1. 颜色模型
   目前常用的颜色模型一种是RGB三原色模型，另外一种广泛采用的颜色模型是亮度、色调、饱和度（IHS颜色模型）。
   亮度表示光谱的整体亮度大小，对应于图像的空间信息属性；

   色调描述纯色的属性，决定于光谱的主波长，是光谱在质的方面的区别；

   饱和度表征光谱的主波长在强度中的比例，色调和饱和度代表图像的光谱分辨率。

   IHS变换图像融合就是建立在IHS空间模型的基础上，其基本思想就是在IHS空间中，将低空间分辨率的多光谱图像的亮度分量用高空间分辨率的灰度图象替换。

2. 基本步骤
   - 将多光谱影像重采样到与全色影像具有相同的分辨率；

   - 将多光谱图像的Ｒ、Ｇ、Ｂ三个波段转换到IHS空间，得到Ｉ、Ｈ、Ｓ三个分量；

   - 以Ｉ分量为参考，对全色影像进行直方图匹配

   - 用全色影像替代Ｉ分量，然后同Ｈ、Ｓ分量一起逆变换到RGB空间，从而得到融合图像。

3. RGB与IHS变换的基本公式
   ![](https://camo.githubusercontent.com/22021142b4edd37de54caa15c21a8861f20ed948/687474703a2f2f7777312e73696e61696d672e636e2f6c617267652f36646562373261336c793166786964356e6b6e63696a323069373037377766352e6a7067)

### 实验代码 ###
- GDALDemo.cpp
```cpp
void IHSImageFusion(float **RGB, float *panBuffTmp, float **IHS, float **resultRGB, int imgXlen, int imgYlen)
{
	/*
		IHS图像融合核心算法
		param: RGB-多光谱图像的RGB值 panBuffTmp-全色图像的像素值 IHS-函数外定义的IHS 
		       resultRGB-存储融合后的图像像素值 imgXlen,imgYlen-图像的长度和宽度	
	*/
	float matrix[3][3] = { { 1.0 / 3.0, 1.0 / 3.0, 1.0 / 3.0 },{ -sqrt(2.0) / 6.0, -sqrt(2.0) / 6.0, sqrt(2.0) / 3.0 },{ 1.0 / sqrt(2.0), -1.0 / sqrt(2.0), 0 } };
	float matrixReverse[3][3] = { { 1.0, -1.0 / sqrt(2.0), 1.0 / sqrt(2.0) },{ 1.0, -1.0 / sqrt(2.0), -1.0 / sqrt(2.0) },{ 1.0, sqrt(2.0), 0 } };
	// RGB2IHS
	for (int m = 0; m < 3; m++)
	{
		for (int n = 0; n < imgXlen*imgYlen; n++)
		{
			IHS[m][n] = 0;	// 初始化为0，函数需要多次调用
			for (int k = 0; k < 3; k++)
			{
				IHS[m][n] += matrix[m][k] * RGB[k][n];
			}
		}
	}
	// 将pan图像数据置换I
	// IHS[0] = panBuffTmp; 指针赋值会导致IHS[0]指向panBuffTmp，下一次调用此函数就会赋值失败
	for (int i = 0; i < imgXlen*imgYlen; i++)
	{
		IHS[0][i] = panBuffTmp[i];
	}
	// 逆变换
	for (int mm = 0; mm < 3; mm++)
	{
		for (int nn = 0; nn < imgXlen*imgYlen; nn++)
		{
			resultRGB[mm][nn] = 0;	// 初始化为0，函数需要多次调用
			for (int kk = 0; kk < 3; kk++)
			{
				resultRGB[mm][nn] += matrixReverse[mm][kk] * IHS[kk][nn];
			}
		}
	}
}

void IHSImageFusionMain(char* srcMULPath, char* srcPANPath, char* resultPath)
{
	/*
		遥感图像融合主函数
		param: 多光谱图像路径，全色图像路径，融合结果图像路径
	*/
	GDALAllRegister();

	GDALDataset* srcMULDS = (GDALDataset*)GDALOpenShared(srcMULPath, GA_ReadOnly);
	GDALDataset* srcPANDS = (GDALDataset*)GDALOpenShared(srcPANPath, GA_ReadOnly);

	int imgMULXlen = srcMULDS->GetRasterXSize();
	int imgMULYlen = srcMULDS->GetRasterYSize();
	int MULBandNum = srcMULDS->GetRasterCount();
	// 创建驱动生成图像
	GDALDataset *resDS = GetGDALDriverManager()->GetDriverByName("GTiff")->Create(resultPath, imgMULXlen, imgMULYlen, MULBandNum, GDT_Byte, NULL);
	
	cout << "MUL image X len: " << imgMULXlen << endl;
	cout << "MUL image Y len: " << imgMULYlen << endl;
	cout << "MUL image Band num : " << MULBandNum << endl;

	int imgPANXlen = srcPANDS->GetRasterXSize();
	int imgPANYlen = srcPANDS->GetRasterYSize();
	int PANBandNum = srcPANDS->GetRasterCount();
	// 用float32类型
	float* panBuffTmp = (float*)CPLMalloc(imgMULXlen*imgMULYlen * sizeof(float));
	srcPANDS->GetRasterBand(PANBandNum)->RasterIO(GF_Read, 0, 0, imgMULXlen, imgMULYlen, panBuffTmp, imgMULXlen, imgMULYlen, GDT_Float32, 0, 0);
	
	cout << "PAN image X len: " << imgPANXlen << endl;
	cout << "PAN image Y len: " << imgPANYlen << endl;
	cout << "PAN image Band num : " << PANBandNum << endl;
	
	// 二维数组，每一个开创数组都是用CPLMalloc，中间的IHS也在此malloc，这样省malloc和free的时间
	float **RGB;
	float **resultRGB;
	float **IHS;
	RGB = (float**)CPLMalloc(MULBandNum * sizeof(float));
	resultRGB = (float**)CPLMalloc(MULBandNum * sizeof(float));
	IHS = (float**)CPLMalloc(MULBandNum * sizeof(float));
	
	// 读取数据到数组中
	for (int i = 0; i < 3; i++)
	{
		RGB[i] = (float*)CPLMalloc(imgMULXlen*imgMULYlen * sizeof(float));
		srcMULDS->GetRasterBand(i + 1)->RasterIO(GF_Read, 0, 0, imgMULXlen, imgMULYlen, RGB[i], imgMULXlen, imgMULYlen, GDT_Float32, 0, 0);	
		resultRGB[i] = (float*)CPLMalloc(imgMULXlen*imgMULYlen * sizeof(float));
		IHS[i] = (float*)CPLMalloc(imgMULXlen*imgMULYlen * sizeof(float));
	}

	// 核心算法直接计算，结果存储在resultRGB中
	IHSImageFusion(RGB, panBuffTmp, IHS, resultRGB, imgMULXlen, imgMULYlen);

	// 直接将float类型像素值放进去
	for (int j = 0; j < 3; j++)
	{
		resDS->GetRasterBand(j + 1)->RasterIO(GF_Write, 0, 0, imgMULXlen, imgMULYlen, resultRGB[j], imgMULXlen, imgMULYlen, GDT_Float32, 0, 0);
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
```
### 实验结果 ###
![](http://donky.top/img/American-fusion.png)
### 实验心得 ###
- 确实融合后清楚多了，真的感觉很神奇
- 喜欢抽象，所以花了很多时间把算法抽象出来，接下来也能用到，这次实验只调用了一次核心算法函数，所以问题没有暴露出来，第六个实验再说。