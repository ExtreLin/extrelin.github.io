---
layout:     post
title:      "OpenGL和CUDA的交互"
date:       2018-05-20 13:00:00
author:     "ExreLin"
header-img: "img/post-bg-opengl.jpg"
catalog: true
tags:
    - 应用
    - OpenGL
---

>“OpenGL和cuda的交互. ”


最近在写OpenGL模拟扫描的算法需要用到OpenGL去渲染很多张图片，然后需要分析这些图片。最初的方案是用OpenGL的glReadPixels去读取渲染的buffer然后再保存在内存中再分析。这个方案主要是glReadPixels这条函数特别的慢，需要CPU等待。后来查了资料又将图片read到PBO(pixels buffer objects)中，因为是GPU到GPU，所以可以减少cpu等待周期，但是最后计算还是要通过map的形式映射到内存，所以觉得很慢。所以在想直接在GPU中完成所有计算只输出一个结果，那么就可以大大提高效率。<br>

直接在GPU中并行计算有很多方式，比如OpenGL的computer shader 亦或CUDA、OpenCL之类的并行库。由于当时想学习CUDA所以就选择了CUDA去做这件事。<br>


cuda中可以注册两种形式的buffer, 一种是image类型的buffer，如renderbuffer或者texture,另一种就是vbo，vertexBufferObject。<br>

>这是一段代码。

```cpp
GLuint rbo,vbo;
cudaGraphicResource_t cudaRbo,cudaVbo;
uint w,h;

//generate buffers
glGenRenderbuffers(1, &rbo);
glRenderbufferStorage(GL_RENDERBUFFER, GL_RGBA, w, h);
glGenBuffer(1, &vbo);
....
//register to cuda
cudaGraphicGLRegisterImage(&cudaRbo, rbo, GL_RENDERBUFFER, cudaGraphicRegisterFlagsReadOnly);
cudaGraphicGLRegisterBuffer(&cudaVbo, vbo, cudaGraphicRegisterFlagsReadOnly);

//map to cuda
cudaGraphicMapResource(1, &cudaRbo, 0);
cudaGraphicMapResource(1, &cudaVbo, 0);

//image to array
cudaArray_t cudaRboArray;
cudaGraphicSubResourceGetMappedArray(&cudaRboArray, cudaRbo, 0, 0);

//convert to your type
unsigned char* cudaColorPtr;
cudaMemcpyFromArray(cudaColorPtr, cudaRboArray, 0 , 0, sizeof(unsigned char)*w*h*4, cudaMemcpyDeviceToDevice);

//get buffer pointer
float* cudaBufferPtr;
cudaGraphicResourceGetMappedPointer((void**)&cudaBufferPtr, sizeof(float)*3, cudaVbo);//only need three float

//获得cuda的指针后就可以在cuda的核函数中使用了。
....

//umap
cudaGraphicUnmapResource(1, &cudaRbo, 0);
cudaGraphicUnmapResource(1, &cudaVbo, 0);

//unregister
cudaGraphicGLUnregisterImage(cudaRbo);
cudaGraphicGLUnregisterBuffer(cudaVbo);

```


> 总结

OpenGL和CUDA的交互主要分为注册、映射、获得GPU指针、释放这几个步骤。将所对应的OpenGL资源注册到CUDA，这样OpenGL更新资源的时候CUDA得到对应的更新，然后通过map来将OpenGL资源映射到CUDA，随后做的修改都能在Unmap的时候更新到OpenGL(当然要在注册的时候指定这块资源是可读可写的才行)。<br>

这篇博客只是简单的介绍了最基本的交互操作，其实还有通过texture的方式去获得图像数据，避免一次GPU的拷贝等等操作，有时间会继续更新。<br>


