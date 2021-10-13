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
