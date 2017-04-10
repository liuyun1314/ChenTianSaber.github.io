---
layout: post
category: article
title: 有关Android截图与录屏功能的学习
tags: ChenTian, article
keywords: ChenTian, chentian,Android,学习笔记
excerpt: 这是我的学习笔记
redirect_from:
  - /2017/4/study notes/
---
#有关Android截图与录屏功能的学习

这篇文章，会带你学习如何使用MediaProjection，MediaCodec以及MediaMuxer来实现简单的截屏和录屏功能。  

因为MediaProjection是5.0以上才出现的，所以今天所讲述功能实现，只在5.0以上的系统有效。  

才疏学浅，讲不深入，见谅。

###截屏：
#####步骤如下：
1：获取MediaProjectionManager  

2：通过MediaProjectionManager.createScreenCaptureIntent()获取Intent  

3：通过startActivityForResult传入Intent然后在onActivityResult中通过MediaProjectionManager.getMediaProjection(resultCode,data)获取MediaProjection  

4：创建ImageReader,构建VirtualDisplay  

5:最后就是通过ImageReader截图，就可以从ImageReader里获得Image对象。  

6:将Image对象转换成bitmap

#####实现：
步骤已经给出了，我们就按照步骤来实现代码吧。

首先MediaProjectionManager是系统服务，我们通过getSystemService(MEDIA_PROJECTION_SERVICE)获取它  

```
projectionManager = (MediaProjectionManager) getSystemService(MEDIA_PROJECTION_SERVICE);
```
然后调用startActivityForResult传入projectionManager.createScreenCaptureIntent()创建的Intent  

```
startActivityForResult(projectionManager.createScreenCaptureIntent(),
                SCREEN_SHOT);
```  

紧接着我们就可以在onActivityResult(int requestCode, int resultCode, Intent data)中通过resultCode和data来获取MediaProjection  

```
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        if(requestCode == SCREEN_SHOT){
            if(resultCode == RESULT_OK){
                //获取MediaProjection
                mediaProjection = projectionManager.getMediaProjection(requestCode,data);
            }
        }
    }
```  

然后就是创建ImageReader和VirtualDisplay  

```
        imageReader = ImageReader.newInstance(width, height, PixelFormat.RGBA_8888, 1);
        if(imageReader!=null){
            Log.d(TAG, "imageReader Successful");
        }
        mediaProjection.createVirtualDisplay("ScreenShout",
                width,height,dpi,
                DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
                imageReader.getSurface(),null,null);
```  

这里我们依次讲解一下。  

######首先是ImageReader.newInstance方法：  

```
public static ImageReader newInstance(int width, int height, int format, int maxImages)
```  

方法里接收四个参数。  

前两个width,height是用来指定生成图像的宽和高。  

第三个参数format是图像的格式，这个格式必须是 [ImageFormat](https://developer.android.google.cn/reference/android/graphics/ImageFormat.html)
 或[PixelFormat](https://developer.android.google.cn/reference/android/graphics/PixelFormat.html)中的一个，这两个Format里有很多格式，大家可以点进去看看，我们例子中使用的是PixelFormat.RGBA_8888格式(需要注意的是并不是所有的格式都被ImageReader支持，比如说ImageFormat.NV21)。
第四个参数是maxImages，这个参数指的是你想同时在ImageReader里获取到的Image对象的个数，这个参数我不是很懂，我不理解同时的意思。我的理解是ImageReader是一个类似数组的东西，然后我们可以通过acquireLatestImage()或acquireNextImage()方法来得到里面的Image对象(可能有误，仅供参考)。这个值应该设置的越小越好，但是得大于0，所以我们上面设置的是1。  

######然后我们看看mediaProjection.createVirtualDisplay方法：  

```
createVirtualDisplay(@NonNull String name,
            int width, int height, int dpi, int flags, @Nullable Surface surface,
            @Nullable VirtualDisplay.Callback callback, @Nullable Handler handler)
```  

首先这个方法返回的是VirtualDisplay。  

前四个不用说了，分别是VirtualDisplay的名字，宽，高和dpi。  

第五个参数，大家可以点 [DisplayManager](https://developer.android.google.cn/reference/android/hardware/display/DisplayManager.html)查看所有的flags，我没有具体的研究过，在本次要实现的例子里，除了VIRTUAL_DISPLAY_FLAG_SECURE这个会报错，其他的flags效果都一样。
第六个参数，是一个Surface。我这里表达一下我的理解，当VirtualDisplay被创建出来时，也就是createVirtualDisplay调用后，你在真实屏幕上的每一帧都会输入到Surface参数里。也就是说，如果你放个SurfaceView，然后传入SurfaceView的Surface那么你在屏幕上的操作都会显示在SurfaceView里(这里我们后面录屏会讲)。我们这里传入的是ImageReader的Surface。这其中的逻辑我的理解是这样的，真实屏幕的每一帧都都会传给ImageReader，根据ImageReader的maxImages参数，比如说maxImages是2，那么ImageReader始终保持两帧图片，但这两帧图片是一直随着真实屏幕的操作而更新的(不知道大家有没有听懂)。  

第七个参数，是一个回调函数，在VirtualDisplay状态改变时调用。因为我们这里没有，所以传null。  

第八个参数，这里我给出原文：“The Handler on which the callback should be invoked, or null if the callback should be invoked on the calling thread's main Looper
.”因为我翻译不好。不过和普通的Handler使用场景类似。  


现在我们ImageReader和VirtualDisplay，接下来我们就可以通过ImageReader的acquireLatestImage()或acquireNextImage()来得到Image对象了。  

```
        SystemClock.sleep(1000);
        Image image = imageReader.acquireNextImage();
```  

这里有个坑，就是你在获取Image的时候，得先暂停1秒左右，不然就会获取失败(原因未知)。

现在我们有了Image对象，但是Image对象并不能直接作为UI资源被使用，我们可以将它转换成Bitmap对象。  

```
        int width = image.getWidth();
        int height = image.getHeight();
        final Image.Plane[] planes = image.getPlanes();
        final ByteBuffer buffer = planes[0].getBuffer();
        int pixelStride = planes[0].getPixelStride();
        int rowStride = planes[0].getRowStride();
        int rowPadding = rowStride - pixelStride * width;
        bitmap = Bitmap.createBitmap(width + rowPadding / pixelStride, height, Bitmap.Config.ARGB_8888);
        bitmap.copyPixelsFromBuffer(buffer);
        image.close();
```  

这里最主要的逻辑就是像素与字节的转换，我们需要将Image对象的字节流写进Bitmap里，但是Bitmap接收的是像素格式的。  

我们一行一行来看：  

首先获取image对象的宽和高，注意width和height是像素格式的。  

然后获取ByteBuffer，里面存放的就是图片的字节流，是字节格式的。我是这么理解的，ByteBuffer里面是一长串的字节序列，按照某种格式分成行列就变成了图片。  

然后获取PixelStride，这指的是两个像素的距离(就是一个像素头部到相邻像素的头部)，这是字节格式的。  

RowStride是一行占用的距离(就是一行像素头部到相邻行像素的头部)，这个大小和width有关，这里需要注意，因为内存对齐的原因，所以每行会有一些空余。这个值也是字节格式的。  

紧接着我们需要创建一个Bitmap用来接受Image的buffer的输入，buffer是字节流，它会按照我们设置的format转换成像素，所以这里最重要的一个地方就是Bitmap创建的大小，因为高度就是行数所以就是height，但是宽度因为上面说的内存对齐问题会有些空余，所以我们要先求出空余部分，然后加上width。  

```
int rowPadding = rowStride - pixelStride * width;
```
这句话用整行的距离减去了一行里像素及空隙占用的距离，剩下的就是空余部分。但是这个是字节格式的。我们将它除以pixelStride，也就是一个像素及空隙占用的字节大小，就转换成了像素格式。  

然后：  

```
width + rowPadding / pixelStride
```  

这个就是一行里像素的占用了，我们将它传给Bitmap：  

```
bitmap = Bitmap.createBitmap(width + rowPadding / pixelStride, height, Bitmap.Config.ARGB_8888);
```  

创建出合适大小的Bitmap，然后把Image的buffer传给它，就成功的将Image对象转换成了Bitmap。  

这里我可能讲的不清楚，我给大家画了张图：  


![IMG_3469.JPG](http://upload-images.jianshu.io/upload_images/2649238-f3d845ab7769b8e1.JPG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  


上面的一小格一小格是一块块像素。  

好了，现在我们已经获取到了bitmap了，我们可以把它放到ImageView里显示一下，我写了一个例子，效果如下：  


![5.gif](http://upload-images.jianshu.io/upload_images/2649238-8f70d6b9511da77c.gif?imageMogr2/auto-orient/strip)  


点击按钮，弹出一个对话框请求截屏，点击立即开始的话，截屏就会显示在下面的ImageView里。  


截屏就这样，我已经尽力了，╮(╯▽╰)╭  


###录屏：
#####步骤：
录屏的前三步和截屏是一样的，出现分歧点的地方在于VirtualDisplay创建时传入的Surface，还记得我们上面说的吗，说在创建VirtualDisplay的时候，传入一个SurfaceView的Surface的话，那么你在真实屏幕上的操作，都会重现在SurfaceView上。我们来试一下：  

```
mediaProjection.createVirtualDisplay("ScreenShout",
                width,height,dpi,
                DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
                surfaceView.getHolder().getSurface(),null,null);
```  

我们在Surface参数中传入一个SurfaceView的Surface
效果如下：  


![6.gif](http://upload-images.jianshu.io/upload_images/2649238-f445bd8250f933d7.gif?imageMogr2/auto-orient/strip)  

可以看到我们放了一个Button，放了一个ImageView，放了一个SurfaceView。  

点击Button，然后点立即开始之后，真实屏幕就映射到了SurfaceView里。  

所以当创建VirtualDisplay时，真实屏幕就映射到了Surface，也就是我们可以再Surface里拿到屏幕的一个输入。那我们要录屏的话，就只要把Surface转换成我们需要的格式就行了，在本篇文章的例子中，我们会将Surface对象转换成mp4格式。这就需要用到MediaCodec类和MediaMuxer类。MediaCodec生成一个Surface用来接收屏幕的输出并按照格式编码，然后传给MediaMuxer用来封装成mp4格式的视频。  


```  
        //第一个参数是mime类型，我们传入video/avc
        //第二第三个参数是宽和高
        MediaFormat format = MediaFormat.createVideoFormat("video/avc", width, height);
        //COLOR_FormatSurface这里表明数据将是一个graphicbuffer元数据
        format.setInteger(MediaFormat.KEY_COLOR_FORMAT,
                MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface);
        //设置码率，码率越大视频越清晰，相对的占用内存也要更大
        format.setInteger(MediaFormat.KEY_BIT_RATE, 6000000);
        //设置帧数
        format.setInteger(MediaFormat.KEY_FRAME_RATE, 30);
        //设置两个关键帧的间隔，这个值你设置成多少对我们这个例子都没啥影响
        //这个值做视频的朋友可能会懂，反正我不是很懂，大概就是你预览的时候，比如你设置为10，那么你10秒内的预览图都是同一张
        format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, 10);
        //创建一个MediaCodec实例
        mediaCodec = MediaCodec.createEncoderByType("video/avc");
        //第一个参数将我们上面设置的format传进去
        //第二个参数是Surface，如果我们需要读取MediaCodec编码后的数据就要传，但我们这里不需要所以传null
        //第三个参数关于加解密的，我们不需要，传null
        //第四个参数是一个确定的标志位，也就是我们现在传的这个
        mediaCodec.configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
        //获取MediaCodec的surface，这个surface其实就是一个入口，屏幕作为输入源就会进入这个入口，然后交给MediaCodec编码
        surface = mediaCodec.createInputSurface();
        mediaCodec.start();
```  

上面讲了MediaCodec的创建，我们也可以从中看到屏幕数据是怎么进入MediaCodec的。具体的我已经注释了。

接下来我们创建一个MediaMuxer对象：  

```
//第一个参数是输出的地址
//第二个参数是输出的格式，我们设置的是mp4格式
mediaMuxer = new MediaMuxer(filePath, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);
```  

然后创建VirtualDisplay，把MediaCodec的surface传进去：  

```
virtualDisplay = mediaProjection.createVirtualDisplay(TAG + "-display",
                            width, height, dpi, DisplayManager.VIRTUAL_DISPLAY_FLAG_PUBLIC,
                            surface, null, null);
```  

最后就是视频的编码与转换MP4还有保存了：  

```
    private void recordVirtualDisplay() {
        while (!mQuit.get()) {
            //dequeueOutputBuffer方法你可以这么理解，它会出列一个输出buffer(你可以理解为一帧画面),返回值是这一帧画面的顺序位置(类似于数组的下标)
            //第二个参数是超时时间，如果超过这个时间了还没成功出列，那么就会跳过这一帧，去出列下一帧，并返回INFO_TRY_AGAIN_LATER标志位
            int index = mediaCodec.dequeueOutputBuffer(bufferInfo, 10000);
            //当格式改变的时候吗，我们需要重新设置格式
            //在本例中，只第一次开始的时候会返回这个值
            if (index == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
                resetOutputFormat();

            } else if (index >= 0) {//这里说明dequeueOutputBuffer执行正常
                //这里执行我们转换成mp4的逻辑
                encodeToVideoTrack(index);
                mediaCodec.releaseOutputBuffer(index, false);
            }
        }
    }
    //这里是将数据传给MediaMuxer，将其转换成mp4
    private void encodeToVideoTrack(int index) {
        //通过index获取到ByteBuffer(可以理解为一帧)
        ByteBuffer encodedData = mediaCodec.getOutputBuffer(index);
        //当bufferInfo返回这个标志位时，就说明已经传完数据了，我们将bufferInfo.size设为0，准备将其回收
        if ((bufferInfo.flags & MediaCodec.BUFFER_FLAG_CODEC_CONFIG) != 0) {
            bufferInfo.size = 0;
        }
        if (bufferInfo.size == 0) {
            encodedData = null;
        } 
        if (encodedData != null) {
            encodedData.position(bufferInfo.offset);//设置我们该从哪个位置读取数据
            encodedData.limit(bufferInfo.offset + bufferInfo.size);//设置我们该读多少数据
            //这里将数据写入
            //第一个参数是每一帧画面要放置的顺序
            //第二个是要写入的数据
            //第三个参数是bufferInfo，这个数据包含的是encodedData的offset和size
            mediaMuxer.writeSampleData(videoTrackIndex, encodedData, bufferInfo);
            
        }
    }

    //这个方法其实就是设置MediaMuxer的Format
    private void resetOutputFormat() {
        //将MediaCodec的Format设置给MediaMuxer
        MediaFormat newFormat = mediaCodec.getOutputFormat();
        //获取videoTrackIndex，这个值是每一帧画面要放置的顺序
        videoTrackIndex = mediaMuxer.addTrack(newFormat);
        mediaMuxer.start();
        muxerStarted = true;
    }
```  

好了，录屏到此结束了。
我们来看下实例演示：

![7.gif](http://upload-images.jianshu.io/upload_images/2649238-0f0c937387053a98.gif?imageMogr2/auto-orient/strip)  



###总结：
这篇博客写的真是费时费力，果然水平未到就不该强行写文。

我不知道我是不是写清楚了，但还是希望大家看了之后能有一丝丝的收获，这就是对我最大的鼓励。

本篇博客的录屏代码参考自：**[ScreenRecorder](https://github.com/yrom/ScreenRecorder)**

本篇博客的实例代码：**[ScreenRecorderShoter](https://github.com/ChenTianSaber/ScreenRecorderShoter)**

####最后：

######如果有问题，请在评论区留言，才疏学浅，欢迎大家批评指正。

####最后的最后：
######感谢我可爱的女朋友。