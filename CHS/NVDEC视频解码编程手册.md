# NVDEC 视频解码器 API 编程手册

## 简介

NVIDIA GPUs，从 NVIDIA® Fermi™ 架构开始搭载了一个视频解码引擎（本文称NVDEC）。这个视频解码引擎提供了完全加速的硬件视频解码能力。NVDEC可以被使用去解码多种格式的比特流，如：AV1, H.264, HEVC (H.265), VP8, VP9, MPEG-1, MPEG-2, MPEG-4 和 VC-1。NVDEC独立于计算和图形引擎运行。

NVIDIA为NVDEC编程提供了软件API和库。软件API（本文称NVDECODE API）让开发者能够使用视频解码功能并且在GPU上和其他引擎一起组合使用。

NVDEC解码压缩视频流并拷贝解压结果——YUV帧到视频内存。视频帧图可以通过CUDA进行后处理。NVDECODE API 也提供了对常见视频输出格式的CUDA优化的后处理操作，例如缩放，裁剪，宽高比转换，反交错和颜色空间转换。客户端可以通过NVDEC API选择使用CUDA优化后的处理步骤，也可以选择编写自己的后处理操作。

解码后的视频帧图可以被用视频回放展示，能够被直接传输到编码器上以获得更强的视频转码能力，也能够被用作视频加速推理，或者进一步的CUDA，CPU处理。

### Supported Codecs

编解码器通过NVDECODE API支持以下格式：
* MPEG-1,
* MPEG-2,
* MPEG4,
* VC-1,
* H.264 (AVCHD) (8 bit),
* H.265 (HEVC) (8bit, 10 bit 和 12 bit),
* VP8,
* VP9(8bit, 10 bit 和 12 bit),
* AV1的Main特性,
* 混合(CUDA + CPU)JPEG
关于视频特征和不同种类GPU的更多细节参见第二章。

## 视频解码能力
表1 展示了不同GPU架构下硬件解码器的解码能力。
表1

## 视频解码管道

解码管道由三个部分组成，Demuxer，Video Parser和Video Decoder。组件之间不相互依赖，因此可以独立使用。NVDECODE API提供NVIDIA video parser和NVIDIA video decoder的API。另外，NVIDIA video parser是纯软件组件。有需要的话，用户可以使用任何parser取代NVIDIA video parser（例如，FFmpeg parser）。

图1 使用NVDECODE API的视频解码器管道

视频解码器管道使用NVDECODE API

简单来说，任何使用NVDECODEAPI去解码视频必须通过以下几个步骤：
1. 创建CUDA上下文.
2. 查询硬件解码器的解码能力。
3. 创建解码器实例.
4. 解复用视频。该步骤可以使用第三方软件，如FFMPEG。
5. 通过NVDECODE API或者第三方软件，如FFmpeg解析视频bit流。
6. 开始使用NVDECODE API解码。
8. 查询已解码帧的状态。
9. 根据解码状态，对解码结果进行进一步处理，如绘制，推理，后处理等。
10. 如果应用需要去展示解码结果，
    * 将解码后的YUV转换为RGBA.
    * 将RGBA surface映射到DirectX或者OpenGL的材质上。
    * 将纹理绘制到平面上。
11. 在完成解码过程后，销毁解码器实例。
12. 销毁CUDA上下文。
之后的文档将会解释以上步骤，并在Video Codec SDK的例程中展示。

## 使用NVIDIA视频解码器（NVDECODE API）
所有NVDECODE API都包括在两个头文件：cuviddec.h和nvcuvid.h。这两个头文件在Video Codec SDK package里面的..\Samples\NvCodec\NvDecoder的文件夹。

### 视频解析器

#### 创建一个解析器
填充CUVIDPARSERPARAMS结构体可以创建解析器对象。结构体应该包括待解码视频流的以下信息。

CodecType：必须是一个枚举对象cudaVideoCodec。描述了编码类型，例如H264，HEVC，VP9等。

ulMaxNumDecodeSurfaces：这个数值代表解码图片缓存中最大的表面数。因为在初始化解析器的时候，这个数值可能是一个模糊的数值，所以这个值可以被设置为1来创建一个解析器对象。应用必须通过驱动注册一个供解析器调用的回调pfnSequenceCallback去计数he first sequence header or any changes in the sequence。这个回调通过CUVIDEOFORMAT::min_num_decode_surfaces告知了最小解析器最小的表面数。在想要更改的时候，回调函数会不断向解析器更新这个数值。当这个值大于1的时候（详见下文关于pfnSequenceCallback的描述），解析器会覆盖覆盖这个值。因此，为了最优化内存使用，解析器对象的创建应该被推迟到知道CUVIDPARSERPARAMS::ulMaxNumDecodeSurfaces之后。这样的话，解码器对象可以用需要的缓存数目创建，使得CUVIDDECODECREATEINFO::ulNumDecodeSurfaces = CUVIDPARSERPARAMS::ulMaxNumDecodeSurfaces。

ulMaxDisplayDelay: 最大显示回调延迟，0 = 没有延迟。
pfnSequenceCallback: 应用必须注册一个函数去处理sequence change，解析器触发这个回调为了初始化sequence header或者当解析器记录到一个格式变化。sequence回调会在以下返回值时被驱动中断：
    0：失败。
    1：成功，但是驱动没有覆盖CUVIDPARSERPARAMS::ulMaxNumDecodeSurfaces。
    \>1: 成功，并且驱动用返回值覆盖CUVIDPARSERPARAMS::ulMaxNumDecodeSurfaces。

pfnDecodePicture：在一帧的比特流数据就绪时，解析器触发这个回调。在field pictures的情况下，每次显示要调用两次解码，因为两个field装饰一帧。回调会在以下返回值时被驱动中断。
    0: 失败。
    ≥1: 成功。

pfnDisplayPicture: 在一帧解码就绪时，解码器触发这个回调。回调会在以下返回值的时候被中断。
    0: 失败。
    ≥1: 成功。

#### 解码Packets

Packets通过cuvidParseVideoData(),从demultiplexer送入解码器。无论是sequence change还是图片准备被解码，或者解码和显示，在cuvidParseVideoData()中同步地创建解析器时触发回调。如果回调返回失败，Packets将会被cuvidParseVideoData()propagate到应用。

解码结果会在CUVIDPICPARAMS结构体中和图片索引一起保存。图片索引会在之后用于映射显存中的解码后的帧。

#### 销毁解码器

用户需要调用cuvidDestoryVideoPaser()去销毁解码器对象并回收所有调用的资源。The user needs to call cuvidDestroyVideoParser() to destroy the parser object and free up all the allocated resources.

### 视频解码器

#### 查询解码能力

通过API，cuvidGetDecoderCaps()，能让用户查询硬件视频解码器的能力。

如表1，不同的GPU解码器，有着不同的硬件解码能力。因此，为了确认你的应用能够工作在每一代GPU硬件上，我们非常建议应用查询硬件解码能力并基于解码能力和功能的有无做出合适的选择

通过API，cuvidGetDecoderCaps()，能让用户查询硬件视频解码器的能力。调用线程会有稳定的CUDA上下文联系。

在调用cuvidGetDecoderCaps()之前，客户端需要如下补充CUVIDDECODECAPS的参数。

eCodecType：Codec类型(AV1, H.264, HEVC, VP9, JPEG etc.)
eChromaFormat：4:2:0，4:4:4等.
nBitDepthMinus8：0对应8-bit, 2对应10-bit, 4对应12-bit
当调用cuvidGetDecoderCaps()时，底层驱动会填充underlying的参数，说明支持的能力，支持的输出格式和硬件支持的最大最小分辨率。。

下面的伪码说明了怎样去查询NVDEC的能力。

```cpp
CUVIDDECODECAPS decodeCaps = {};
// set IN params for decodeCaps
decodeCaps.eCodecType = cudaVideoCodec_HEVC;//HEVC 
decodeCaps.eChromaFormat = cudaVideoChromaFormat_420;//YUV 4:2:0
decodeCaps.nBitDepthMinus8 = 2;// 10 bit
result = cuvidGetDecoderCaps(&decodeCaps);
```

Returned parameters from API can be interpreted as below to validate if content can be decoded on underlying hardware:

```cpp
// Check if content is supported
if (!decodecaps.bIsSupported){
        NVDEC_THROW_ERROR(Codec not supported on this GPU", CUDA_ERROR_NOT_SUPPORTED);
}
// validate the content resolution supported on underlying hardware
if ((coded_width > decodecaps.nMaxWidth) ||
        (coded_height > decodecaps.nMaxHeight)){
        NVDEC_THROW_ERROR(Resolution not supported on this GPU", CUDA_ERROR_NOT_SUPPORTED);
}
// Max supported macroblock count CodedWidth*CodedHeight/256 must be <= nMaxMBCount
if ((coded_width>>4)*(coded_height>>4) > decodecaps.nMaxMBCount){
       NVDEC_THROW_ERROR(MBCount not supported on this GPU",
CUDA_ERROR_NOT_SUPPORTED);
}
```

In most situations, bit-depth and chroma subsampling to be used at the decoder output is same as that at the decoder input (i.e. in the content). In certain cases, however, it may be necessary to have the decoder produce output with bit-depth and chroma subsampling different from that used in the input bitstream. In general, it’s always a good idea to first check if the desired output bit-depth and chroma subsampling format is supported before creating the decoder. This can be done in the following way:

```cpp
// Check supported output format
if (decodecaps.nOutputFormatMask & (1<<cudaVideoSurfaceFormat_NV12)){
        // Decoder supports output surface format NV12
}
if (decodecaps.nOutputFormatMask & (1<<cudaVideoSurfaceFormat_P010){
        // Decoder supports output surface format P010
}
```

The API cuvidGetDecoderCaps() also returns histogram related capabilities of underlying GPU. Histogram data is collected by NVDEC during the decoding process resulting in zero performance penalty. NVDEC computes the histogram data for only the luma component of decoded output, not on post-processed frame(i.e. when scaling, cropping, etc. applied). In case of AV1 when film gain is enabled, histogram data is collected on the decoded frame prior to the application of the flim grain.

```cpp
// Check if histogram is supported
if (decodecaps.bIsHistogramSupported){
        nCounterBitDepth = decodecaps.nCounterBitDepth; // histogram counter bit depth
        nMaxHistogramBins = decodecaps.nMaxHistogramBins; // Max number of histogram bins
}
```

Histogram data is calculated as : Histogram_Bin[pixel_value >> (pixel_bitDepth - log2(nMaxHistogramBins))]++;

## 译者附录
部分翻译索引
Paser：解析器

未翻译英文解释


## 部分相关知识
