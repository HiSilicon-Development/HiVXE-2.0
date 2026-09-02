# HiSilicon Hi3798Cv200 HiVXE 2.0 Decoding & Transcoding

HiVXE 2.0 是一份面向 HiSilicon Hi3798Cv200 的公共 HiVXE 编解码能力说明，列出当前 Linux/FFmpeg 已支持的接口、仍未开放的 BSP 能力，以及对应的 FFmpeg 9.0.1 运行时。

内容以 VDEC/VENC 为主，同时说明与它配合、但硬件上相对独立的 JPGD/JPGE 和 VPSS/MCDEI。

本文描述的是 Hi3798Cv200 公共编解码能力，重点是接口、数据格式和能力边界，不把某一块板卡的软件包装方式当成编解码能力本身。

## 状态

| 状态 | 含义 |
| --- | --- |
| ✅ | 当前 Linux/FFmpeg 已有实现，公开接口和运行路径已有验证依据；不代表所有组合都已覆盖 |
| 🟨 | 已有部分接口或实现依据，但还缺语法、生命周期、样片或组合覆盖 |
| 🟦 | 原厂 BSP 的生产配置、可达解析器/硬件抽象层或完整调用链有依据；当前 Linux 尚未移植或完成验证 |
| ❌ | 当前公开接口没有提供，或原厂产品路径已明确拒绝 |

`🟦 原厂 BSP 依据` 不是“隐藏的现成功能”。在建立可维护的公开接口、输出缓冲格式、错误处理和真实设备验证以前，它不能作为安装、发布或应用程序可用性的承诺。

解码、编码和后处理章节中的表格以当前公开 Linux/FFmpeg 接口为主，状态表示这条公共路径目前能承诺到什么程度。`✅` 表示实现和运行路径已有验证依据，`🟨` 表示路径存在但组合覆盖仍有限，`🟦` 只表示原厂 BSP 的历史依据，`❌` 表示当前公开路径不提供该能力。

## 组件说明

| 组件 | 作用 |
| --- | --- |
| VDEC / VDH | HiVXE 中负责压缩视频重建的硬件解码部分 |
| VENC | H.264 硬件编码部分 |
| VPSS / MCDEI | 缩放、去隔行、降噪等视频后处理，不等同于某个解码器 |
| JPGD / JPGE | 独立 JPEG 解码/编码模块，不与 VDEC 共用同一接口 |
| V4L2 Request | 用户态按帧提交码流、语法控件和参考帧的 Linux 无状态解码接口 |
| V4L2 M2M | Linux 的内存到内存编码/转换接口 |
| BSP | 海思原厂板级软件与私有驱动；有 BSP 实现不等于当前开源 Linux 已可使用 |

## 当前可用路径

| 路径 | 接口或组件 | 当前用途 |
| --- | --- | --- |
| CPU 软件路径 | FFmpeg 软件解码器，例如 `cavs` | 不依赖 HiVXE 硬件节点，作为默认路径和硬件协商失败时的 fallback |
| V4L2 Request | 无状态解码控件、参考帧和输出队列 | 按帧提交码流并使用 HiVXE VDEC/VDH；具体格式以解码矩阵为准 |
| V4L2 M2M | `h264_v4l2m2m` | 使用 HiVXE VENC 做 H.264 4:2:0 编码 |
| JPGD/JPGE | 独立 JPEG 接口 | JPEG 编解码，不与 VDEC Request 节点混用 |
| VPSS/MCDEI | `histb-vpss` 及原厂后处理链 | 当前 Linux 主要承担 VDEC 的 tile-to-linear；独立 MCDEI 能力仍按 BSP 状态说明 |

## 解码

本节按压缩格式列出当前 Linux/FFmpeg 解码路径的输入、语法控件、参考帧和输出限制。表中的 `✅`、`🟨`、`🟦` 和 `❌` 不是芯片规格等级，而是公开接口在这些条件下能否被维护和复现的结论。

### H.264 / AVC

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | 8-bit 4:2:0 Baseline/Main/High | CPU 解码和 V4L2 Request 解码均保留；输出为 NV12 或软件帧 |
| ✅ | I/P/B、CABAC、CAVLC | 逐帧 Request 控件和参考列表由用户态提交 |
| ✅ | 加权预测 | P/B 的亮度、色度权重和偏移进入解码路径 |
| ✅ | 多 slice、DPB、显示重排 | 支持多 slice 和长期/完整参考列表 |
| ✅ | MBAFF | 输出完整 4:2:0 帧，不提供单场像素格式 |
| 🟨 | 独立 field picture | Linux 已有上下场配对和消息打包，但仍缺独立 field 专用样片矩阵 |
| 🟦 | 8-bit monochrome（`chroma_format_idc=0`） | 原厂 CV200 SPS 解析器接受；当前 Linux 缺单平面输出缓冲格式、接口约定和样片 |
| ❌ | High 10、High 4:2:2、High 4:4:4 | 当前输出缓冲格式和 V4L2 接口约定固定为 8-bit 4:2:0；原厂接口也把这些 profile 标为不支持 |
| ❌ | FMO、redundant picture、data partition、SVC | 没有对应的消息和参考层接口 |

### HEVC / H.265

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | Main 8-bit 4:2:0 | VPS/SPS/PPS、I/P/B、参考列表和显示顺序已接入 |
| ✅ | Main 10 4:2:0 / P010 | 10-bit 输出缓冲格式和 Request 控件可用 |
| ✅ | 多 slice、dependent slice、tile、WPP | 按 CTB 行和熵同步边界提交 |
| 🟨 | PCM/IPCM、SAO、deblock | 基础消息字段已提供，极端组合仍需独立格式矩阵 |
| 🟨 | transform skip、transquant bypass、加权预测 | 控件和消息入口存在，组合覆盖仍有限 |
| ❌ | 4:0:0、4:2:2、4:4:4、12-bit、SCC/RExt | 当前输出缓冲格式和产品格式校验不提供这些组合 |
| 🟨 | Dolby Vision 双层 | BSP 头部保留条件结构，但没有已证明的生产调用链；当前 Linux 缺双层参考和输出接口约定 |

### MPEG-1 / MPEG-2

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | MPEG-2 Main 4:2:0 I/P/B | sequence、picture、量化矩阵、field 和参考列表均有 Request 控件 |
| 🟨 | MPEG-2 Simple 4:2:0 I/P | BSP 和 Linux 校验允许该 profile，但仍缺独立 Simple Profile 样片矩阵 |
| 🟨 | MPEG-1 4:2:0 I/P/B | 独立 Request 控件和匹配内核/模块已构建；含 full-pel 的硬件路径仍需设备验证 |
| ✅ | field picture、显示重排、EOS/drain | 输出完整 NV12 帧，不暴露单场格式 |
| ❌ | 4:2:2、spatial/SNR scalable | 当前产品输出缓冲格式和消息路径不提供 |

### MPEG-4 Part 2

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | SP/ASP 8-bit 4:2:0 I/P/B | VOL/VOP、resync、video packet 和 packed bitstream 可处理 |
| ✅ | QPEL、GMC、S-VOP、uncoded VOP | 受硬件消息容量和输入尺寸边界约束 |
| 🟨 | interlaced VOL/VOP | 解析器和消息字段存在，需按具体工具组合验证 |
| ❌ | data partitioning、NEWPRED、reduced-resolution、static sprite | 当前编解码接口约定明确拒绝 |

### VC-1

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | Advanced/Main 的常规 I/P/B | progressive、frame-interlace 和 field-interlace 路径已拆分 |
| 🟨 | Simple Profile | BSP 与 Linux 解析器有 Simple/Main 路径，但仍缺独立样片和完整生命周期矩阵 |
| ✅ | BPD、参考帧、intensity compensation | 参考输出缓冲、bitplane 和显示规划由同一 Request 生命周期管理 |
| ✅ | RESPIC coded geometry | 按当前 coded macroblock 网格计算 BPD 和重建区域 |
| 🟨 | range-reduction、interlaced SKIP 的全部组合 | 入口和校验已存在，极端组合仍需更多格式向量 |
| ❌ | 硬件 repair | 默认禁用；没有安全、完整的 repair DMA 与错误恢复接口约定，不作为可用功能 |
| ❌ | 不受支持的私有扩展 | 不根据寄存器预留位推断额外编解码能力 |

### VP8 / VP9

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | VP8 8-bit Frame、token partition、segmentation | hidden/altref、概率状态和参考刷新有对应路径 |
| ✅ | VP9 Profile 0、8-bit 4:2:0 | frame context、loopfilter、show-existing 和 multi-tile 已接入 |
| 🟦 | VP9 Profile 2、10-bit 4:2:0 / P010 | BSP 启用 10-bit，消息含位深与采样字段；当前 Linux 仍只接受 Profile 0/8-bit |
| 🟨 | VP9 Profile 1/3、4:2:2/4:4:4 | BSP 解析器能读取相关语法，但原生输出缓冲和后处理链依据不足；当前 Linux 不支持 |
| ❌ | VP9 12-bit | 当前产品位深边界不提供 |

### AVS / AVS+

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | AVS Baseline（profile 0x20） | CPU 软件解码和 V4L2 Request 硬件解码均有路径 |
| ✅ | AVS+（profile 0x48）渐进 I/P/B | AEC、参考列表、slice 链和显示重排已接入 |
| ✅ | AVS+ field picture、P/B field-enhanced | CPU 与 Request 使用相同的 picture flags 和参考语义 |
| ✅ | picture-level PAWQ | 使用标准的单张 8×8 weighting matrix |
| ✅ | P/B weighted prediction | slice 权重和宏块选择位由 CPU/AVSP 解码路径处理 |
| 🟨 | low-delay 与 sequence boundary | 解析和同尺寸 epoch 切换已提供；改变尺寸时 Request 需要重新协商队列 |
| ❌ | “MBAWQ”双矩阵模式 | AVS+ 语法中没有该模式；WQ 后的单 bit 是 reserved，不能当作第二矩阵选择位 |
| ❌ | AVS+ 4:2:2 | 当前 Hi3798Cv200 产品路径只提供 4:2:0 重建输出，不能仅放开 header 位 |
| ❌ | 动态 profile/宽高无停流切换 | 需要结束旧 sequence、排空参考帧并重新建立 Request 捕获缓冲池 |

当前接口使用 `V4L2_PIX_FMT_AVS_SLICE` 以及 sequence、picture、slice、decode 四组无状态控制。

驱动还依赖大小固定为 17,920 字节的 `hisilicon/histb-avsp.bin`；固件缺失、尺寸不符或 DMA 地址不满足 CV200 VDH 约束时，请求应在启动硬件前失败。

BSP 中的 AEC、PAWQ、weighted prediction 和 field 状态机为这些公开控件提供了来源依据，但并不会自动扩大 Linux 当前的 profile、色度或动态重配边界。

### 原厂 BSP 有依据、但当前 Linux 未开放的格式

这些条目用于说明 Hi3798Cv200 原厂软件曾覆盖到哪里，不表示当前 Linux 已提供对应节点，也不自动构成后续移植目标。

下表只摘录能够追溯到 R002 SPC030 生产配置、可达解析器、硬件抽象层或调用链的项目。配置对象、枚举和编译产物不等于公开 ABI、固件或当前运行接口。

| 当前 Linux 状态 | 格式 | 原厂 BSP 依据 |
| --- | --- | --- |
| ❌ | MVC | R002 私有源码中有解析器、subset-SPS 及双视角枚举/数据结构；当前 Linux 缺双视角 DPB、inter-view reference、消息和左右眼输出接口约定 |
| ❌ | RealVideo 8/9/10 | R002 生产配置选择 RV8/RV30、RV9/10/RV40 解析器与专用硬件抽象层对象；没有公开输入接口、参考帧和输出路径 |
| ❌ | VP6 | R002 生产配置选择 VP6 解析器与 VDH 硬件抽象层对象；当前 Linux 没有输入接口，历史 Flash 视频格式不作为新增目标 |
| ❌ | DivX 3 | 配置与能力表有 DivX3 条目，但已检查的 decode/start/write 路径是 no-op/stub；不能据此宣称 BSP 硬件可用 |
| ❌ | H.263 | R002 `SOS_CFG` 明确关闭；`HD_FULL` 仅在 SVDEC 条件下开启；配置变体不足以证明生产路径 |
| ❌ | JPEG/MJPEG 解码 | BSP 有独立 JPGD 解析器/硬件抽象层和 4:4:4 图像路径；当前 Linux 未公开 JPGD 解码接口，它也不是 VDEC Request 节点 |
| ❌ | PNG 解码 | BSP 有 libpng 与 TDE 显示链；没有与 JPGD/VDH 同等级的专用消息、IRQ 和硬件解码链依据 |

AVS2 与上述“BSP 有实现证据”的项目不同：CV200 源码虽保留条件结构，生产 `vfmw_config.cfg` 未启用 AVS2，当前 Linux 也没有输入 ABI 或硬件消息路径。

## 编码

当前公开编码路径以 H.264 V4L2 M2M 和 baseline JPEG 为主。编码表中的限制同时包含硬件产品边界、公开控件和输出缓冲格式，不能仅依据原厂枚举或寄存器名称扩大能力。

### H.264

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | 8-bit 4:2:0 NV12 I/P 编码 | Linux V4L2 M2M 接口，输出 Annex-B |
| ✅ | CBR、固定 QP、GOP、强制关键帧 | 参数越界返回错误，不静默放宽限制 |
| 🟨 | Main/High 头字段和 CABAC | 编码接口已提供，完整 profile 组合仍需独立验证 |
| ❌ | B 帧、ROI、OSD | 当前没有公开的参考重排或控制接口约定 |
| ❌ | HEVC 编码 | 当前 Hi3798Cv200 VENC 路径不提供 |

### JPEG

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | Baseline JFIF/MJPEG、NV12 4:2:0 输入 | 使用独立 JPEG encoder 接口，不与 HiVXE VDEC 混用 |
| 🟦 | 4:2:2、4:4:4、packed/planar 输入 | 原厂 JPGE 定义这些布局；当前 Linux 尚未移植格式协商和地址/stride 打包 |
| ❌ | 旋转与 slice 编码 | CV200 原厂 JPGE 产品检查明确拒绝，当前 Linux 也不公开这些控制 |
| ❌ | Progressive/lossless JPEG 编码 | 当前 Linux encoder 只公开 baseline 输出 |

## 后处理

后处理需要和解码器输出、帧存布局及转码阶段分开阅读。当前 Linux 已使用的部分、CPU 滤镜和只在原厂 BSP 中出现的 VPSS/MCDEI 路径分别标注，三者不互相等价。

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | VDEC tile-to-linear | 当前 `histb-vpss` 在 VDEC 内部完成 tile 数据面到线性帧的转换，不提供独立通用节点 |
| 🟨 | CPU bob/反交错 | `hivxebob`/`bwdif` 在 CPU 上处理，不应写成 VPSS/MCDEI 硬件能力 |
| 🟦 | 独立 VPSS/MCDEI 去隔行 | BSP 有四场 RGME、运动统计、历史帧写回、blend 和 init/complete/reset 链；当前 Linux 无公共接口和场生命周期 |
| 🟦 | TNR/SNR/MCNR | BSP 有时域/空域降噪寄存器、统计 buffer 及与 MCDEI 的连接；当前 Linux 未移植算法参数和用户接口 |
| 🟦 | 旋转与多输出端口 | 原厂 VPSS 有旋转输出和多 port 调度；当前 Linux 只把后处理用于 VDEC 内部路径 |

所以“解码器能输出隔行内容”“CPU 滤镜能 bob”和“独立 MCDEI 硬件已开放”是三件不同的事。

只有第三项建立公开 Linux 接口并通过场序、运动、错误恢复和吞吐验证后，才能从 `🟦` 变为当前可用功能。

## 接口与转码选择

FFmpeg 未指定 `-hwaccel` 时默认走 CPU 软件解码，输出通常是 `yuv420p`。显式指定 `-hwaccel v4l2request` 才会尝试 HiVXE Request；硬件初始化或格式协商失败时，FFmpeg 可能回退到 CPU，应从日志确认最终选择的格式。

`/dev/mediaX` 是命令模板中的设备节点，应按系统实际枚举结果替换。

## Tvheadend Spawn profile 模板

这些命令把解码、下载为线性 NV12、CPU 反交错、H.264 编码和 MPEG-TS 输出串成一条可用于 Tvheadend 的 Spawn 管线。

Tvheadend 不会自动生成本页中的 profile；需要转码时，先在 `tvheadend-spawn-profile` 中手动新增一个 MPEG-TS Spawn profile，再把一整行命令填入命令字段。

`hivxe-1080p50-bob` 和 `hivxe-720p50-bob` 只是便于识别的名称，不代表已有同名文件。

### Request 解码 + H.264 编码：1080p profile

```text
/usr/local/bin/hivxe-ffmpeg -hide_banner -loglevel warning -nostdin -xerror -init_hw_device v4l2request=hivxe:/dev/mediaX -hwaccel v4l2request -hwaccel_device hivxe -hwaccel_output_format drm_prime -threads 1 -extra_hw_frames 0 -f mpegts -probesize 1048576 -analyzeduration 2000000 -fflags +genpts+discardcorrupt -i pipe:0 -map 0:v:0 -map 0:a? -map 0:s? -vf hwdownload,format=nv12,hivxebob -c:v h264_v4l2m2m -b:v 6000000 -g 25 -c:a copy -c:s copy -fps_mode passthrough -muxdelay 0 -muxpreload 0 -flush_packets 1 -mpegts_flags +resend_headers -f mpegts pipe:1
```

### Request 解码 + H.264 编码：720p profile

```text
/usr/local/bin/hivxe-ffmpeg -hide_banner -loglevel warning -nostdin -init_hw_device v4l2request=hivxe:/dev/mediaX -hwaccel v4l2request -hwaccel_device hivxe -hwaccel_output_format drm_prime -threads 1 -extra_hw_frames 0 -f mpegts -probesize 1048576 -analyzeduration 2000000 -fflags +genpts+discardcorrupt -i pipe:0 -map 0:v:0 -map 0:a? -map 0:s? -vf hwdownload,format=nv12,scale=1280:720:flags=fast_bilinear:interl=1,format=nv12,hivxebob,format=nv12 -c:v h264_v4l2m2m -num_capture_buffers 16 -b:v 6000000 -g 50 -c:a copy -c:s copy -fps_mode passthrough -muxdelay 0 -muxpreload 0 -flush_packets 1 -mpegts_flags +resend_headers -f mpegts pipe:1
```

### CPU 解码 + H.264 编码

没有硬件解码条件时，把 `-c:v cavs` 保留在输入侧即可；编码仍可使用 V4L2 M2M：

```text
/usr/local/bin/hivxe-ffmpeg -hide_banner -loglevel warning -nostdin -xerror -f mpegts -probesize 1048576 -analyzeduration 2000000 -fflags +genpts+discardcorrupt -c:v cavs -i pipe:0 -map 0:v:0 -map 0:a? -map 0:s? -vf format=nv12,hivxebob -c:v h264_v4l2m2m -b:v 6000000 -g 50 -c:a copy -c:s copy -fps_mode passthrough -muxdelay 0 -muxpreload 0 -flush_packets 1 -mpegts_flags +resend_headers -f mpegts pipe:1
```

### 某些源的强制反交错

有些源的码流标志写成 progressive，但画面实际由交错场组成。对这类输入，不要依赖自动判断，解码后明确使用：

```text
bwdif=mode=send_field:parity=auto:deint=all
```

`deint=all` 强制处理每个输入帧，`parity=auto` 只选择场序，`mode=send_field`
为每个场输出一个时间点。因此 25 帧/秒的隔行输入可以形成名义 50p 的输出。
反交错是 CPU 后处理，不是 AVS/AVS+ 解码器本身的格式开关。需要硬件解码时，
把它放在 `hwdownload,format=nv12` 之后；只用 CPU 解码时，直接放在
`-c:v cavs` 的输出滤镜链中。

### 参数说明

| 参数或处理段 | 作用 |
| --- | --- |
| `-c:v cavs` | 选择 AVS/AVS+ CPU 软件解码；不带 `-hwaccel` 时不会启用 HiVXE Request |
| `-init_hw_device ...`、`-hwaccel v4l2request` | 初始化并选择 V4L2 Request 硬件解码设备 |
| `-hwaccel_output_format drm_prime` | 让解码器输出 DRM PRIME 硬件帧，后续需要 `hwdownload` 才能进入 CPU 滤镜 |
| `hwdownload,format=nv12` | 将硬件帧下载为线性 NV12 |
| `hivxebob` | 按场输出渐进帧；它在 CPU 上执行，不是 VDEC 寄存器功能 |
| `-c:v h264_v4l2m2m` | 选择 HiVXE H.264 V4L2 M2M 编码器 |
| `-b:v 6000000`、`-g 25`/`-g 50` | 设置目标码率和 GOP 长度；`-g` 不是输出帧率 |
| `-map 0:a?`、`-map 0:s?` | 音频或字幕存在时复制，不存在时不使命令失败 |
| `-fps_mode passthrough` | 保留滤镜产生的时间戳，不额外重复或丢弃帧 |
| `-muxdelay 0 -muxpreload 0 -flush_packets 1` | 减少 MPEG-TS 复用缓存并及时输出 |
| `-mpegts_flags +resend_headers` | 在输出 TS 中重复发送表信息，便于重新锁定频道 |

### 错误写法与误读

- Tvheadend Spawn 字段直接拆分参数，命令必须是一行；不要写成 shell 脚本，也不要使用 `...`、变量替换、重定向、管道或 `&&`。输入和输出应保持为 `pipe:0`、`pipe:1`。
- 不要把 `-g 25` 或 `-g 50` 当作帧率；它们只表示 GOP 长度。
- 不要删除可选流映射末尾的 `?`，否则没有音频或字幕的频道可能因映射失败退出。
- `-xerror` 只把 FFmpeg 判定为错误的处理失败转成退出状态，不会把所有驱动 warning 自动变成错误；错误日志和队列积压仍需单独检查。
- 对实际交错画面不要写 `bwdif=deint=interlaced`；progressive 标记会使它跳过画面，这里应保留 `deint=all`。
- `-fps_mode passthrough` 和输出元数据中的 `50/1` 只描述时间戳，不保证整条解码、反交错、编码链达到实时速度。

## 解读边界

- 解码器输出格式以当前 V4L2/FFmpeg 接口约定为准；芯片规格或旧软件列出的分辨率、帧率不自动扩大 Linux 的可接受格式。
- 语法字段、寄存器预留位或旧格式名称本身不构成支持证据；必须同时有可达解析器、消息布局和输出缓冲格式。
- BSP 的能力开关也不单独构成运行证据；必须排除空函数、stub、条件未编译和只存在产品 ID 映射的情况。
- CPU 软件路径与 Request 硬件路径是独立选择。未指定硬件 flag 时应保持 CPU fallback；指定硬件但协商失败时应明确记录回退原因。
- 反交错、缩放和编码属于后处理/转码阶段，不能把它们的命令行选项写成 AVS/AVS+ 解码能力。

## 证据与来源

本文按四类来源判断能力边界：

- **公开规格资料**只说明芯片设计目标，不证明 Linux 接口已经实现；
- **原厂 BSP**必须同时看到生产配置和可达解析器、硬件抽象层或调用链，才可标为 `🟦`；
- **当前 Linux/FFmpeg**必须有公开 ABI、构建闭包和可复现的格式、错误及重开验证，才可标为 `✅`；
- **运行结果**必须和内核、模块、固件、用户态及样片版本绑定，不能跨版本外推。

BSP 条目来自 Hi3798Cv200 R002 SPC030（Android 5.1.1、Linux 3.18）研究材料中的 `vfmw_config.cfg`、受功能开关控制的解析器/硬件抽象层、VENC v1.0、JPGD/JPGE 和 VPSS v4.0 路径。

可公开核对的参考包括：

- [`histb-mainline`](https://github.com/histb-mainline) 及其 [Linux 内核树](https://github.com/histb-mainline/linux)，用于核对公开主线化工作的范围；
- [HiSilicon-Development/kernel](https://github.com/HiSilicon-Development/kernel)，用于核对本项目当前公开 Linux 接口；
- [HiSTBLinux R005 SPC050](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050) 与 [SPC060](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060)，仅用于比较较晚的公开 SDK 结构，不作为 R002 精确证据。

本文只保留可公开说明的 BSP 摘要，没有复制专有源码、二进制、固件、私有接口、本机绝对路径或过时的结论。
