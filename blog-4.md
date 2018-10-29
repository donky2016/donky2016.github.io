## 线性滤波 ##
-----
### 相关知识 ###
1. 基本概念
线性滤波可以说是图像处理最基本的方法，它可以允许我们对图像进行处理，产生很多不同的效果。做法非常简单，首先，有一个二维的滤波器矩阵（卷积核）和一个待处理的二维图像。然后，对于图像中每一个像素点，计算它的领域像素和滤波器矩阵对应元素的乘积，然后加起来，作为该像素位置的值，这样就完成了滤波过程。
2. 卷积计算
以一个像素点为中心，取与卷积核相同大小的领域，并让相同位置上的像素值相乘之后加在一起，就是这个像素点的卷积和，将每一个像素都计算一遍，获的卷积运算的结果
3. 卷积核不同，运算出的图像也会不同，本次实现了几种：
   - 均值模糊
   - 运动模糊
   - 边缘检测
   - 锐化
   - 浮雕
   - 高斯模糊
  
    **具体的滤波器及其参数在代码注释中体现**
4. 滤波器的规则
   - 滤波器的大小应该是奇数，这样它才有一个中心，例如3X3、5X5，或者7X7。
   - 滤波器所有元素之和应该是1，这是为了保持滤波前后图像的亮度保持不变。
   - 如果滤波矩阵所有元素之和大于 1，那么滤波后的图像就会比原图像亮。反之，如果小于1，那么得到的图像就会变暗。
   - 对于滤波后的结构，可能会出现负数或者大于255的数值，对于这种情况，我们将他们直接截断至0和255之间即可。
### 代码实现 ###
- **为了以后项目的进行，这里直接写了两个函数，可以直接输入图像路径及卷积核参数就能够自动化生成滤波后的图像**
- GDALDemo.cpp (part)
    ```c++
    GByte* getConvolutionSum(GByte* bufftemp, int Xlen, int Ylen, double convolutionKernel[], int kernelLen, int param, int offset)
    {
        /*
            获取所有像素的卷积和的值
            params: 源图像数据，原图像的X，Y，和卷积核，卷积核的长度(宽度)，卷积和需要除的数，和偏移量
            return: 经过卷积和计算的图像数据GByte类型的指针
        */
        GByte* sumData = (GByte*)CPLMalloc(Xlen*Ylen * sizeof(GByte));
        double tmp = 0;		// 存储每一个像素的卷积和
        for (int i = 0; i < Xlen; i++)
        {
            for (int j = 0; j < Ylen; j++)
            {
                tmp = 0;
                // 筛选出边界并置0
                if ((i == 0) ||  (i == Xlen - 1) || (j == 0) || (j == Ylen - 1))
                {
                    sumData[i*Ylen + j] = tmp;
                }
                else
                {
                    for (int m = 0; m < kernelLen; m++)
                    {
                        for (int n = 0; n < kernelLen; n++)
                        {
                            /*
                                卷积核中心坐标是[(kernelLen - 1) / 2, (kernelLen - 1) / 2]
                                卷积核中任一坐标为(m,n)，则(XDiff,YDiff)为相对中心位置
                                根据相对位置，就能算出图像中任一像素的邻域像素值
                            */
                            int XDiff = n - (kernelLen - 1) / 2 ;
                            int YDiff = m - (kernelLen - 1) / 2 ;
                            tmp += convolutionKernel[m*kernelLen+n] * bufftemp[i*Ylen + j + XDiff*Ylen + YDiff];
                        }					
                    }
                    tmp = tmp / param + offset;
                    // 截断，如果像素值不在（0,255）之间
                    if (tmp > 255)
                    {
                        sumData[i*Ylen + j] = 255;
                    }
                    else if (tmp < 0)
                    {
                        sumData[i*Ylen + j] = 0;
                    }
                    else
                    {
                        sumData[i*Ylen + j] = (GByte) round(tmp);
                    }
                }

            }
        }
        return sumData;
    }
    void convolutionConvert(char* poSrcPath, char* poResultPath, double convolutionKernel[], int kernelLen, int param, int offset)
    {
        /*
            根据原图像，卷积核，卷积核的长度(宽度)，卷积和需要除的数，和偏移量得到转化之后的图像
        */
        GDALAllRegister();

        GDALDataset* poSrcDS = (GDALDataset*) GDALOpenShared(poSrcPath, GA_ReadOnly);
        
        cout << "Loading lean.jpg..." << endl;

        int imgXlen = poSrcDS->GetRasterXSize();
        int imgYlen = poSrcDS->GetRasterYSize();
        int bandNum = poSrcDS->GetRasterCount();
        cout << "X len: " << imgXlen << endl;
        cout << "Y len: " << imgYlen << endl;
        cout << "Band num : " << bandNum << endl;
        
        GDALDataset* resultDS = GetGDALDriverManager()->GetDriverByName("GTiff")->Create(poResultPath, imgXlen, imgYlen, bandNum, GDT_Byte, NULL);
        GByte* bufftmpSrc = (GByte*)CPLMalloc(imgXlen*imgYlen * sizeof(GByte));
        GByte* bufftmpResult = (GByte*)CPLMalloc(imgXlen*imgYlen * sizeof(GByte));

        
        for (int b = 0; b < bandNum; b++)
        {
            poSrcDS->GetRasterBand(b + 1)->RasterIO(GF_Read, 0, 0, imgXlen, imgYlen, bufftmpSrc, imgXlen, imgYlen, GDT_Byte, 0, 0);
            bufftmpResult = getConvolutionSum(bufftmpSrc, imgXlen, imgYlen, convolutionKernel, kernelLen, param, offset);
            resultDS->GetRasterBand(b + 1)->RasterIO(GF_Write, 0, 0, imgXlen, imgYlen, bufftmpResult, imgXlen, imgYlen, GDT_Byte, 0, 0);
        }

        CPLFree(bufftmpSrc);
        CPLFree(bufftmpResult);
        GDALClose(poSrcDS);
        GDALClose(resultDS);
    }

    void convolutionMain()
    {
        /*
            线性滤波主函数，根据不同卷积核获取不同的图像	
        */
        char* poSrcPath = "Data\\pictures\\lena.jpg";
        char* resultPath = "Data\\output\\convolution.jpg";
        int kernekLen = 0;

        /*  卷积核1  均值模糊
            0 , 1 ,  0
            1 , 1 ,  1		/ 5  （param=5）
            0 , 1 ,  0
        */
        resultPath = "Data\\output\\convolution-1.jpg";
        double convolutionKernel_1[9] = { 0, 1, 0, 1, 1, 1, 0, 1, 0 };
        kernekLen = sqrt(sizeof(convolutionKernel_1) / sizeof(double));
        convolutionConvert(poSrcPath, resultPath, convolutionKernel_1, kernekLen, 5, 0);

        /*  卷积核2  运动模糊
            1, 0, 0, 0, 0,
            0, 1, 0, 0, 0,
            0, 0, 1, 0, 0,    / 5	(param=5)
            0, 0, 0, 1, 0,
            0, 0, 0, 0, 1,
        */
        resultPath = "Data\\output\\convolution-2.jpg";
        double convolutionKernel_2[25] = { 
            1,0,0,0,0,
            0,1,0,0,0,
            0,0,1,0,0,
            0,0,0,1,0,
            0,0,0,0,1 };
        kernekLen = sqrt(sizeof(convolutionKernel_2) / sizeof(double));
        convolutionConvert(poSrcPath, resultPath, convolutionKernel_2, kernekLen, 5, 0);
        
        /* 卷积核3 边缘检测
            -1,-1,-1,
            -1, 8,-1,
            -1,-1,-1,
            
        */
        resultPath = "Data\\output\\convolution-3.jpg";
        double convolutionKernel_3[9] = { -1,-1,-1,-1,8,-1,-1,-1,-1 };
        kernekLen = sqrt(sizeof(convolutionKernel_3) / sizeof(double));
        convolutionConvert(poSrcPath, resultPath, convolutionKernel_3, kernekLen, 1, 0);

        /* 卷积核4  锐化
            -1,-1,-1,
            -1, 9,-1,
            -1,-1,-1,		
        */
        resultPath = "Data\\output\\convolution-4.jpg";
        double convolutionKernel_4[9] = { -1,-1,-1,-1,9,-1,-1,-1,-1 };
        kernekLen = sqrt(sizeof(convolutionKernel_4) / sizeof(double));
        convolutionConvert(poSrcPath, resultPath, convolutionKernel_4, kernekLen, 1, 0);
        /* 卷积核5  浮雕
            -1,-1, 0,
            -1, 0, 1,		offset=128
            0, 1, 1,
        */
        resultPath = "Data\\output\\convolution-5.jpg";
        double convolutionKernel_5[9] = { -1,-1,0,-1,0,1,0,1,1 };
        kernekLen = sqrt(sizeof(convolutionKernel_5) / sizeof(double));
        convolutionConvert(poSrcPath, resultPath, convolutionKernel_5, kernekLen, 1, 128);
        /* 卷积核6 高斯模糊
            0.0120,0.1253,0.2736,0.1253,0.0120,
            0.1253,1.3054,2.8514,1.3054,0.1253,
            0.2736,2.8514,6.2279,2.8514,0.2736,
            0.1253,1.3054,2.8514,1.3054,0.1253,
            0.0120,0.1253,0.2736,0.1253,0.0120,
        */
        resultPath = "Data\\output\\convolution-6.jpg";
        double convolutionKernel_6[25] = {
            0.0120,0.1253,0.2736,0.1253,0.0120,
            0.1253,1.3054,2.8514,1.3054,0.1253,
            0.2736,2.8514,6.2279,2.8514,0.2736,
            0.1253,1.3054,2.8514,1.3054,0.1253,
            0.0120,0.1253,0.2736,0.1253,0.0120
        };
        kernekLen = sqrt(sizeof(convolutionKernel_6) / sizeof(double));
        convolutionConvert(poSrcPath, resultPath, convolutionKernel_6, kernekLen, 25, 0);
    }

    int main()
    {
        convolutionMain();
        system("pause");
        return 0;
    }
    ```
### 结果 ###
- 原图 lena.jpg
  ![](http://ww1.sinaimg.cn/large/006B2c7cly1fwpl5x9651j30740740ty.jpg)
- 均值模糊
  ![](http://ww1.sinaimg.cn/large/006B2c7cly1fwpl88uz13j3074074dg2.jpg)
- 运动模糊
  ![](http://ww1.sinaimg.cn/large/006B2c7cly1fwpla64ugdj3074074q34.jpg)
- 边缘检测
  ![](http://ww1.sinaimg.cn/large/006B2c7cly1fwplbcnsi6j307407474y.jpg)
- 锐化
  ![](http://ww1.sinaimg.cn/large/006B2c7cly1fwplbw96oyj30740740tm.jpg)
- 浮雕
  ![](http://ww1.sinaimg.cn/large/006B2c7cly1fwplcnpfhcj3074074aap.jpg)
- 高斯模糊
  ![](http://ww1.sinaimg.cn/large/006B2c7cly1fwplcynbaxj3074074jrl.jpg)
  