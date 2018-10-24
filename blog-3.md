## GDAL图像合成实现（抠像） ##
-----
### 相关知识 ###

- 前期知识已足够
- 使用颜色取样器，找到不需要的部分图像的RGB值的范围

### 实现代码 ###

- GDALDemo.cpp（part）

  ```c++
  /*
  	图像合成函数，将superman图像得绿色背景扣掉，把superman嵌入space图像上
  	方式：
  		1.将两个图像得三个通道分别存储在supermanBufftmp[3]，spaceBufftmp[3]中
  		2.比较superman中像素值，将绿背景去掉，其他像素值赋值给space图像中相同得位置
  		3.将spaceBufftemp数据写入结果图像，生成合成图像
  */
  void imageSynthesis()
  {
  	GDALDataset* supermanSrcDS;
  	GDALDataset* spaceSrcDS;
  	GDALDataset* resultDS;
  
  	char* supermanPath = "Data\\pictures\\superman.jpg";
  	char* spacePath = "Data\\pictures\\space.jpg";
  	char* supermanAndSpacePath = "Data\\output\\superman-on-the-space.tif";
  
  	int imgXlen, imgYlen, bandNum;
  
  	GByte RValue, GValue, BValue;
  
  	GDALAllRegister();
  
  	supermanSrcDS = (GDALDataset*)GDALOpenShared(supermanPath, GA_ReadOnly);
  	spaceSrcDS = (GDALDataset*)GDALOpenShared(spacePath, GA_ReadOnly);
  
  	imgXlen = spaceSrcDS->GetRasterXSize();	
  	imgYlen = spaceSrcDS->GetRasterYSize();		
  	bandNum = spaceSrcDS->GetRasterCount();
  
  	GByte* supermanBufftmp[3];		// bandNum数为3
  	GByte* spaceBufftmp[3];
  	// 图片数据写入bufftmp
  	for (int n = 0; n < bandNum; n++)
  	{
  		supermanBufftmp[n] = (GByte*)CPLMalloc(imgXlen*imgYlen * sizeof(GByte));
  		supermanSrcDS->GetRasterBand(n + 1)->RasterIO(GF_Read, 0, 0, imgXlen, imgYlen, supermanBufftmp[n], imgXlen, imgYlen, GDT_Byte, 0, 0);
  		spaceBufftmp[n] = (GByte*)CPLMalloc(imgXlen*imgYlen * sizeof(GByte));
  		spaceSrcDS->GetRasterBand(n + 1)->RasterIO(GF_Read, 0, 0, imgXlen, imgYlen, spaceBufftmp[n], imgXlen, imgYlen, GDT_Byte, 0, 0);
  	}
  	// 将superman中除绿背景得像素写到space上
  	int flag;
  	for (int i = 0; i < imgXlen; i++)
  	{
  		for (int j = 0; j < imgYlen; j++)
  		{
  			flag = 1;
  			RValue = (GByte)supermanBufftmp[0][i*imgYlen + j];
  			GValue = (GByte)supermanBufftmp[1][i*imgYlen + j];
  			BValue = (GByte)supermanBufftmp[2][i*imgYlen + j];
  			// 将绿背景范围除掉
  			if (RValue > 50 && RValue < 160 && GValue >130 && GValue < 220 && BValue >50 && BValue < 150)
  			{
  				flag = 0;
  			}
  			if (flag)
  			{
  				spaceBufftmp[0][i*imgYlen + j] = RValue;
  				spaceBufftmp[1][i*imgYlen + j] = GValue;
  				spaceBufftmp[2][i*imgYlen + j] = BValue;
  			}
  		}
  	}
  	// 加载驱动
  	resultDS = GetGDALDriverManager()->GetDriverByName("GTiff")->Create(supermanAndSpacePath, imgXlen, imgYlen, bandNum, GDT_Byte, NULL);
  	// 写入合成图像并释放
  	for (int m = 0; m < 3; m++)
  	{
  		resultDS->GetRasterBand(m + 1)->RasterIO(GF_Write, 0, 0, imgXlen, imgYlen, spaceBufftmp[m], imgXlen, imgYlen, GDT_Byte, 0, 0);
  		CPLFree(supermanBufftmp[m]);
  		CPLFree(spaceBufftmp[m]);
  	}
  	GDALClose(supermanSrcDS);
  	GDALClose(spaceSrcDS);
  	GDALClose(resultDS);
  }		
  
  ```

- 结果截图  
	![](http://ww1.sinaimg.cn/large/006B2c7cly1fwhx0konruj30hs0dcjs6.jpg)

