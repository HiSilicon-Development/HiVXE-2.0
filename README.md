# HiSilicon Hi3798CV200 HiVXE 编解码能力

本文先从Hi3798CV200 BSP的产品配置、解析器、HAL和实际调用链确定硬件能力，再与本仓库Linux 6.12.y驱动及FFmpeg接口逐项对照。解码与编码分开，profile、位深、色度、帧类型和工具分别列行。

`Linux 6.12.y`指本项目带HiVXE驱动补丁的内核，不是原生上游6.12内核。能力矩阵描述源码和公开硬件接口；具体安装包可能裁剪FFmpeg组件，不能把CPU软件解码或头文件枚举当成硬件支持。

## 状态

| 状态 | BSP列 | Linux 6.12.y列 |
| --- | --- | --- |
| ✅ 已做 | 匹配CV200的产品配置及可达解析器/HAL或调用链已有实现 | 本仓库已实现并开放对应硬件接口 |
| 🟨 未做／未完成 | 只有部分实现、条件配置，或尚未证明完整可用路径 | 尚未移植、尚未开放，或只完成部分路径；详见各项说明 |
| ❌ 不支持 | 当前CV200产品配置明确排除、资格检查拒绝，或没有可用硬件实现依据 | 当前产品或已开放格式合同不支持，或明确拒绝该输入/功能 |

两列独立判断。BSP已做而Linux未做，不表示有隐藏开关可以直接启用；部分实现也不能当成完整profile支持。这里的“不支持”限定当前CV200产品及本项目接口，不推断其他SoC或芯片物理上绝无可能。✅表示实现状态，不表示所有样片、尺寸、profile工具组合或持续实时吞吐均已验收。CPU软件路径不改变硬件列状态。

## 组件说明

| 组件 | 作用 |
| --- | --- |
| VDEC / VDH | HiVXE 中负责压缩视频重建的硬件解码部分 |
| VENC | CV200 的 H.264 硬件编码部分；生产配置没有 H.265/AVS 编码器 |
| VPSS / MCDEI | 缩放、去隔行、降噪等视频后处理，不等同于某个解码器 |
| JPGD / JPGE | 独立 JPEG 解码/编码模块，不与 VDEC 共用同一接口 |
| V4L2 Request | 用户态按帧提交码流、语法控件和参考帧的 Linux 无状态解码接口 |
| V4L2 M2M | Linux 的内存到内存编码/转换接口 |
| BSP | 海思原厂板级软件与私有驱动；有 BSP 实现不等于当前开源 Linux 已可使用 |

## 当前可用路径

| 路径 | 接口或组件 | 当前用途 |
| --- | --- | --- |
| CPU 软件路径 | FFmpeg 软件解码器，例如 `cavs` | 可单独选择的软件路径；不是固定 `hivxe-transcode` 硬件滤镜链的自动回退 |
| V4L2 Request | 无状态解码控件、参考帧和输出队列 | 按帧提交码流并使用 HiVXE VDEC/VDH；具体格式以解码矩阵为准 |
| Stateful V4L2 M2M解码 | `V4L2_PIX_FMT_MPEG4`、`VC1_ANNEX_G/L` | 内核解析压缩码流并管理显示/参考状态；不是Request控件 |
| V4L2 M2M | `h264_v4l2m2m` | 使用 HiVXE VENC 做 H.264 4:2:0 编码 |
| JPGE | 独立 JPEG V4L2 M2M 接口 | JPEG 编码，不与 VDEC Request 节点混用 |
| JPGD | 原厂独立 JPEG 解码模块 | 当前 Linux 尚无 JPGD 节点，不能借用 JPGE 或 VDEC 节点解码 JPEG |
| VPSS/MCDEI | `histb-vpss` 及原厂后处理链 | 当前 Linux 承担 VDEC 的 tile-to-linear、ZME 下采样和可选 field-rate DEI；完整 MCDI/TNR 仍未开放 |

## 解码

BSP依据以R005 SPC060的Hi3798CV200产品配置及其实际选中的语法/HAL为主。各项工具仍受所属profile约束；例如解码B图像不表示Baseline允许B slice，更不表示VENC能编码B帧。

Linux的H.264、HEVC、MPEG-2、VP8、VP9和AVS/AVS+使用V4L2 Request；MPEG-4 Part 2和VC-1使用stateful V4L2 M2M。CPU软件解码独立存在，表格不把它当成硬件能力。

### H.264 / AVC

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Baseline：8-bit 4:2:0 | ✅ | ✅ |
| Main：8-bit 4:2:0 | ✅ | ✅ |
| High：8-bit 4:2:0 | ✅ | ✅ |
| Constrained Baseline独立约束覆盖 | 🟨 | 🟨 |
| Extended（profile 88） | 🟨 | ❌ |
| High 10（profile 110） | 🟨 | ❌ |
| High 4:2:2 / High 4:4:4 | ❌ | ❌ |
| SVC Scalable Baseline / High（83/86） | 🟨 | ❌ |
| I图像 | ✅ | ✅ |
| P图像 | ✅ | ✅ |
| B图像 | ✅ | ✅ |
| CAVLC | ✅ | ✅ |
| CABAC | ✅ | ✅ |
| P显式加权预测 | ✅ | ✅ |
| B显式加权预测 | ✅ | ✅ |
| B隐式加权预测 | ✅ | ✅ |
| 多slice | ✅ | ✅ |
| 长期参考与DPB | ✅ | ✅ |
| MBAFF | ✅ | ✅ |
| 独立场图像 | ✅ | ✅ |
| 8×8变换与scaling lists | ✅ | ✅ |
| 去块滤波控制 | ✅ | ✅ |
| 8-bit monochrome | 🟨 | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| Baseline：8-bit 4:2:0 | CPU解码与V4L2 Request解码均有路径；硬件输出NV12。Baseline不包含后面明确拒绝的FMO等工具。 |
| Main：8-bit 4:2:0 | BSP和Linux有profile 77及对应语法/参考帧路径；与编码器Main支持分开判断。 |
| High：8-bit 4:2:0 | BSP和Linux有profile 100、8×8变换及scaling-list消息；不是High 10/4:2:2/4:4:4。 |
| Constrained Baseline独立约束覆盖 | 属于Baseline的约束子集，Linux按profile_idc=66处理；未据此声明独立约束识别及完整conformance。与VENC未开放Constrained Baseline菜单分开判断。 |
| Extended（profile 88） | BSP保留尝试解码的子集路径，遇data partition等工具仍受限；Linux公开校验不接收profile 88。不是VENC的Extended支持。 |
| High 10（profile 110） | BSP接纳profile ID、解析位深并报告非8-bit事件，不等于已证明10-bit重建/输出；不能把告警写成parser立即硬拒绝。Linux明确只接受8-bit。 |
| High 4:2:2 / High 4:4:4 | BSP SPS拒绝chroma_format_idc>1，Linux也只接受4:2:0；profile名称或ID不足以扩大色度合同。 |
| SVC Scalable Baseline / High（83/86） | BSP有SPS扩展解析及部分失败分支，但未证明完整分层重建/输出；Linux没有公开SVC链路。 |
| I图像 | 按I slice提交；IDR及普通I图像的参考状态由DPB管理。 |
| P图像 | L0参考列表和运动预测进入VDH消息。 |
| B图像 | L0/L1参考列表与显示重排都有路径；不能据此声称VENC能编码B帧。 |
| CAVLC | PPS熵编码标志为0；包含已修复MB-pair终止地址的MBAFF路径。 |
| CABAC | PPS entropy flag、cabac_init_idc及CABAC硬件数据表已接入。 |
| P显式加权预测 | 亮度/色度权重、偏移和分母写入slice消息。 |
| B显式加权预测 | weighted_bipred_idc=1，分别传递L0/L1权重。 |
| B隐式加权预测 | weighted_bipred_idc=2交给VDH；实现不等于每种组合已实测。 |
| 多slice | 按编码顺序提交，要求first_mb递增；不承诺任意slice乱序。 |
| 长期参考与DPB | Linux验证VALID/ACTIVE/LONG_TERM/FIELD；用户态管理MMCO和显示重排。 |
| MBAFF | 按MB-pair打包，输出完整woven NV12帧。 |
| 独立场图像 | 上下场配对、底场地址与参考语义已有实现；独立field样片覆盖不等于MBAFF验收。 |
| 8×8变换与scaling lists | PPS transform_8x8及量化矩阵都有实际消息写入。 |
| 去块滤波控制 | disable_deblocking_filter_idc、alpha/beta偏移写入slice消息。 |
| 8-bit monochrome | BSP SPS接受chroma_format_idc=0，但完整单平面重建/后处理输出尚未证明；Linux只接受chroma_format_idc=1。 |

当前Linux不支持High 10、High 4:2:2、High 4:4:4、Extended、FMO、redundant picture、data partition、SP/SI slice和完整SVC分层解码。BSP的尝试解析路径不等于这些完整profile可用；CV200 SPS明确拒绝transform bypass。MVC另列，不从profile ID推断Linux双视角能力。

### HEVC / H.265

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Main：8-bit 4:2:0 | ✅ | ✅ |
| Main 10：10-bit 4:2:0 | ✅ | ✅ |
| Main Still Picture | 🟨 | 🟨 |
| I slice / IRAP图像 | ✅ | ✅ |
| P slice | ✅ | ✅ |
| B slice | ✅ | ✅ |
| 独立slice segment | ✅ | ✅ |
| Dependent slice segment | ✅ | ✅ |
| Tiles | ✅ | ✅ |
| WPP | ✅ | ✅ |
| PCM / IPCM | ✅ | ✅ |
| SAO | ✅ | ✅ |
| 去块滤波控制 | ✅ | ✅ |
| Transform skip | ✅ | ✅ |
| Transquant bypass | ✅ | ✅ |
| P加权预测 | ✅ | ✅ |
| B加权预测 | ✅ | ✅ |
| Scaling lists | ✅ | ✅ |
| 长期参考与temporal MVP | ✅ | ✅ |
| Dolby Vision双层完整解码/合成 | 🟨 | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| Main：8-bit 4:2:0 | VPS/SPS/PPS、参考图像和Request消息已有路径，硬件捕获为NV12。 |
| Main 10：10-bit 4:2:0 | CV200真实10-bit重建路径；Linux支持P010，或由VPSS转8-bit NV12。 |
| Main Still Picture | BSP及用户态有PTL解析，内核复用8-bit IRAP/I工具路径；Request SPS控件不携带profile_idc，未证明profile 3的独立完整覆盖。 |
| I slice / IRAP图像 | 当前Linux要求I图像为IRAP；不承诺非IRAP的intra-only图像位置。 |
| P slice | L0参考、POC和运动矢量地址已有实际打包。 |
| B slice | L0/L1参考和显示重排已有实际打包。 |
| 独立slice segment | 码流边界、CTB地址、entry offsets与slice链均受校验。 |
| Dependent slice segment | 启用PPS继承语义，首segment不得为dependent。 |
| Tiles | 有tile几何和tile-scan消息；CV200路径上限为10列、11行。 |
| WPP | entropy_coding_sync启用位、CTB行入口和硬件消息已接入。 |
| PCM / IPCM | 位深、PCM块大小和loop-filter-disable都有校验/消息路径；最大PCM块不超过min(CTB,32)。 |
| SAO | SPS及slice的亮度/色度SAO开关进入硬件。 |
| 去块滤波控制 | slice disable、beta/tc偏移及跨tile/slice策略进入硬件。 |
| Transform skip | 基础HEVC工具有真实PPS到VDH消息路径；不是完整RExt能力。 |
| Transquant bypass | 基础PPS bypass位写入硬件；不扩大位深、色度或完整lossless profile承诺。 |
| P加权预测 | PPS使能和逐参考亮度/色度权重均已打包。 |
| B加权预测 | 两套参考权重均有实际消息路径。 |
| Scaling lists | 4×4至32×32矩阵及DC值写入硬件量化表。 |
| 长期参考与temporal MVP | 长期参考集合、collocated PMV和DPB校验都有路径。 |
| Dolby Vision双层完整解码/合成 | BSP有实际VES splitter、metadata与BL/EL私有策略，不足以证明双层重建/合成；Linux没有双层DPB/输出接口。 |

CV200实际SPS要求4:2:0，luma位深上限10-bit；不提供4:0:0、4:2:2、4:4:4、12-bit、完整RExt或SCC。HDR/Dolby元数据处理不等于双层重建，更不等于HDR tone mapping。

### MPEG-1

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| 8-bit 4:2:0 | ✅ | 🟨 |
| I图像 | ✅ | 🟨 |
| P图像 | ✅ | 🟨 |
| B图像 | ✅ | 🟨 |

| 支持的能力 | 说明 |
| --- | --- |
| 8-bit 4:2:0 | BSP MPEG-2解析器/HAL内有MPEG-1分支；本项目Linux不开放此格式，未将DC-only诊断输出视作成功。 |
| I图像 | BSP有图像与slice消息；Linux不提供MPEG-1硬件接口。 |
| P图像 | BSP有前向参考与full-pel语义；Linux不提供MPEG-1硬件接口。 |
| B图像 | BSP有双向参考语义；Linux不提供MPEG-1硬件接口。 |

### MPEG-2

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Simple Profile：8-bit 4:2:0 | ✅ | ✅ |
| Main Profile：8-bit 4:2:0 | ✅ | ✅ |
| High Profile的4:2:0子集 | 🟨 | 🟨 |
| I图像 | ✅ | ✅ |
| P图像 | ✅ | ✅ |
| B图像 | ✅ | ✅ |
| 逐行帧 | ✅ | ✅ |
| 独立场图像 | ✅ | ✅ |
| 量化矩阵与扫描顺序 | ✅ | ✅ |
| 显示重排与EOS/drain | ✅ | ✅ |

| 支持的能力 | 说明 |
| --- | --- |
| Simple Profile：8-bit 4:2:0 | 只允许I/P；BSP/Linux均区分profile限制。 |
| Main Profile：8-bit 4:2:0 | 公开Request路径，sequence/picture/quantisation及参考帧均有控件。 |
| High Profile的4:2:0子集 | Linux validator接受profile 1且固定chroma_format=1；不据此宣称完整High或4:2:2支持，BSP的生产profile检查以Simple/Main为主。 |
| I图像 | 独立picture/slice及量化矩阵消息。 |
| P图像 | 前向参考消息，符合Simple/Main限制。 |
| B图像 | 用于Main，Simple Profile明确拒绝B图像。 |
| 逐行帧 | progressive_sequence与picture flags一致性受校验。 |
| 独立场图像 | 上下场配对后输出完整NV12帧，不暴露单场像素格式。 |
| 量化矩阵与扫描顺序 | intra/non-intra矩阵、alternate scan及picture级工具进入消息。 |
| 显示重排与EOS/drain | 参考图像与捕获图像分别持有，排空剩余输出。 |

当前Linux只接受非scalable的4:2:0子集；4:2:2、spatial/SNR scalable不能由profile枚举或扩展头函数名推断为支持。

### MPEG-4 Part 2

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Simple Profile：8-bit 4:2:0 | 🟨 | 🟨 |
| Advanced Simple Profile：8-bit 4:2:0 | 🟨 | 🟨 |
| I-VOP | ✅ | ✅ |
| P-VOP | ✅ | ✅ |
| B-VOP | ✅ | ✅ |
| QPEL | ✅ | ✅ |
| GMC / S-VOP | ✅ | ✅ |
| Uncoded VOP | 🟨 | ❌ |
| Resync / video packets | ✅ | ✅ |
| Packed bitstream | ✅ | ✅ |
| 隔行VOL/VOP | ✅ | ✅ |

| 支持的能力 | 说明 |
| --- | --- |
| Simple Profile：8-bit 4:2:0 | BSP和Linux都仅实现工具子集，不能按profile名称声称完整conformance；Linux使用stateful V4L2 M2M。 |
| Advanced Simple Profile：8-bit 4:2:0 | BSP和Linux都仅实现工具子集，不能按profile名称声称完整conformance；Linux使用stateful V4L2 M2M。 |
| I-VOP | VOL/VOP、量化及slice消息已有路径。 |
| P-VOP | 前向参考与显示规划已有路径。 |
| B-VOP | 双向参考与显示重排已有路径。 |
| QPEL | quarter_sample有解析与运动补偿消息。 |
| GMC / S-VOP | 支持动态sprite/GMC，不等同于static sprite。 |
| Uncoded VOP | Linux stateful validator拒绝uncoded/N-VOP；BSP完整复用输出覆盖未单独闭合。 |
| Resync / video packets | packet头与slice边界有真实解析和消息链。 |
| Packed bitstream | stateful解析器可处理同包多VOP；仍受边界和队列校验限制。 |
| 隔行VOL/VOP | 场序与alternate vertical scan进入消息；工具组合仍需样片覆盖。 |

当前Linux不支持complexity estimation、data partitioning、NEWPRED、reduced-resolution、scalability、OBMC和static sprite；GMC/S-VOP只对应动态sprite，仍拒绝sprite brightness-change。

### VC-1 / WMV9

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Simple Profile | ✅ | ✅ |
| Main Profile | ✅ | ✅ |
| Advanced Profile | ✅ | 🟨 |
| 逐行图像 | ✅ | ✅ |
| Advanced frame-interlace I/P/B | ✅ | ✅ |
| Advanced frame-interlace BI/SKIP | 🟨 | ❌ |
| Advanced field-interlace | ✅ | ✅ |
| 逐行B / BI图像 | ✅ | ✅ |
| Bitplane / BPD | ✅ | ✅ |
| Intensity compensation | ✅ | ✅ |
| RESPIC | ✅ | ✅ |
| Main range reduction | ✅ | ✅ |
| Advanced range mapping | ✅ | ✅ |
| PSF | 🟨 | ❌ |
| 场slice重复picture header | 🟨 | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| Simple Profile | Linux Annex L解析profile 0，仅接受符合Simple限制的逐行工具组合。 |
| Main Profile | Linux Annex L解析profile 1且要求RES_RTM=1；不是V4L2 Request。 |
| Advanced Profile | Annex G的主要帧/场路径已实现，但PSF、隔行BI/SKIP和重复场slice头仍被拒绝，不能承诺完整profile。 |
| 逐行图像 | Simple/Main/Advanced分别遵循各自语法限制。 |
| Advanced frame-interlace I/P/B | 参考、bitplane和帧编码几何独立处理。 |
| Advanced frame-interlace BI/SKIP | Linux明确拒绝隔行frame BI及非逐行SKIP；BSP变体覆盖未单独闭合。 |
| Advanced field-interlace | 首场、次场、场事务及参考/输出生命周期有独立实现。 |
| 逐行B / BI图像 | 支持相应profile允许的图像类型；Simple不允许违反其B帧限制。 |
| Bitplane / BPD | bitplane解析、DMA、UP完成校验和参考语义都有路径。 |
| Intensity compensation | 参考映射、按parity组合和DMA表打包都有实现。 |
| RESPIC | 按当前coded macroblock网格计算BPD与重建区域。 |
| Main range reduction | Main语法及参考表面处理有路径。 |
| Advanced range mapping | 有捕获后range-map helper；Linux在CPU执行该后处理，不是纯硬件range mapping。 |
| PSF | Linux Advanced sequence parser明确拒绝PSF；BSP完整输出变体未证明。 |
| 场slice重复picture header | Linux拒绝field slice picture_header_flag=1；不要与已实现的独立场路径混淆。 |

### VP8

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Version 0 | ✅ | ✅ |
| Version 1 | ✅ | ✅ |
| Version 2 | ✅ | ✅ |
| Version 3 | ✅ | ✅ |
| 8-bit 4:2:0 | ✅ | ✅ |
| Segmentation | ✅ | ✅ |
| 1/2/4/8 token partitions | ✅ | ✅ |
| LAST / GOLDEN / ALTREF | ✅ | ✅ |

| 支持的能力 | 说明 |
| --- | --- |
| Version 0 | 普通loop filter与six-tap运动补偿路径。 |
| Version 1 | simple loop filter与bilinear运动补偿路径。 |
| Version 2 | 无loop filter，bilinear运动补偿路径。 |
| Version 3 | 无loop filter，含full-pixel运动补偿选择。 |
| 8-bit 4:2:0 | VP8 Frame Request控件，捕获为NV12。 |
| Segmentation | segment map、量化和滤波调整有实际消息。 |
| 1/2/4/8 token partitions | 分区地址及边界有校验；不是多路并发转码承诺。 |
| LAST / GOLDEN / ALTREF | 参考地址、更新状态和hidden/altref图像生命周期已有路径。 |

版本4–7的experimental字段和默认处理不是独立版本支持保证；不据token partition数量声称多路并发或软件线程数。

### VP9

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Profile 0：8-bit 4:2:0 | ✅ | ✅ |
| Profile 1：8-bit非4:2:0 | ❌ | ❌ |
| Profile 2：10-bit 4:2:0 | 🟨 | ❌ |
| Profile 2：12-bit 4:2:0 | ❌ | ❌ |
| Profile 3：高位深非4:2:0 | ❌ | ❌ |
| Frame context与概率更新 | ✅ | ✅ |
| Loop filter与segmentation | ✅ | ✅ |
| Tiles | ✅ | ✅ |
| Show-existing frame | ✅ | ✅ |

| 支持的能力 | 说明 |
| --- | --- |
| Profile 0：8-bit 4:2:0 | Linux严格接受profile=0、bit_depth=8及双向subsampling，捕获为NV12。 |
| Profile 1：8-bit非4:2:0 | 泛型parser字段不构成CV200非4:2:0重建合同。 |
| Profile 2：10-bit 4:2:0 | BSP能解析profile/depth，但匹配VP9 HAL固定8-bit重建配置；全局10BIT开关不是完整10-bit VP9证明。 |
| Profile 2：12-bit 4:2:0 | 没有匹配的12-bit重建表面和后处理路径。 |
| Profile 3：高位深非4:2:0 | 没有匹配的高位深非4:2:0重建路径。 |
| Frame context与概率更新 | 上下文概率表、参考刷新和错误隔离已有实际路径。 |
| Loop filter与segmentation | 量化、滤波、segment map与硬件概率消息已有实现。 |
| Tiles | tile入口和边界进入消息，格式限制仍为Profile 0。 |
| Show-existing frame | 使用已有参考图像的显示路径；不等同于新的硬件重建job。 |

### AVS / AVS+

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| AVS JiZhun Profile（0x20） | ✅ | ✅ |
| AVS+ Guangdian Profile（0x48） | ✅ | ✅ |
| 8-bit 4:2:0 | ✅ | ✅ |
| I图像 | ✅ | ✅ |
| P图像 | ✅ | ✅ |
| B图像 | ✅ | ✅ |
| 逐行图像 | ✅ | ✅ |
| 场编码图像 | ✅ | ✅ |
| AVS+ AEC | ✅ | ✅ |
| AVS+ PAWQ | ✅ | ✅ |
| AVS+ chroma QP adjustment | ✅ | ✅ |
| AVS+ P-field enhancement | ✅ | ✅ |
| AVS+ B-field enhancement | ✅ | ✅ |
| 独立加权预测 | 🟨 | 🟨 |
| 4:2:2 | 🟨 | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| AVS JiZhun Profile（0x20） | CPU cavs与V4L2 Request均有路径；不要与FFmpeg的无关avs解码器混淆。 |
| AVS+ Guangdian Profile（0x48） | 生产配置开启AVSPLUS，AVSP与VDH消息已有实现。 |
| 8-bit 4:2:0 | Linux要求sample_precision=1、chroma_format=1，输出NV12。 |
| I图像 | sequence/picture/slice/decode四组Request控件。 |
| P图像 | 前向参考及显示重排有路径。 |
| B图像 | 双向参考及显示重排有路径。 |
| 逐行图像 | picture structure和progressive flags受校验。 |
| 场编码图像 | 两场必须合并为完整picture提交；MPEG-TS需启用cavsvideo parser。 |
| AVS+ AEC | AVSP固件与对应picture标志/熵编码语法均已接入。 |
| AVS+ PAWQ | picture-level单张8×8 weighting matrix；属于加权量化，不是加权预测。 |
| AVS+ chroma QP adjustment | 色度QP偏移进入picture消息。 |
| AVS+ P-field enhancement | 仅允许匹配AVS+ P场图像的增强标志。 |
| AVS+ B-field enhancement | 仅允许匹配AVS+ B场图像的增强标志。 |
| 独立加权预测 | 不能从PAWQ矩阵或AVSP存在推断完整P/B加权预测覆盖；本次未找到独立完整合同。 |
| 4:2:2 | BSP保留chroma字段，但未证明完整CV200重建输出；Linux明确只接受4:2:0。 |

未证明10/12-bit、4:4:4完整重建或无停流profile/宽高切换。PAWQ是单张量化矩阵；不能把reserved位解释成“MBAWQ双矩阵”。

### MVC（H.264多视角扩展）

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Stereo High（profile 128）双视角 | ✅ | 🟨 |
| Multiview High（profile 118）双视角 | ✅ | 🟨 |

| 支持的能力 | 说明 |
| --- | --- |
| Stereo High（profile 128）双视角 | BSP subset-SPS、inter-view reference及H.264 HAL消息可达；Linux没有公开MVC前端/双视角输出。 |
| Multiview High（profile 118）双视角 | CV200 BSP最多两个视角；不承诺任意N视角。 |

CV200 subset-SPS明确限制最多两个视角。Linux虽接受118/128的profile ID，但当前构建不链接独立MVC实现，也未接通inter-view参考管理和双视角输出；旧构建对象不构成支持。

### RealVideo

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| RealVideo 8 / RV30 | ✅ | 🟨 |
| RealVideo 9 / RV40 | ✅ | 🟨 |
| RealVideo 10独立变体覆盖 | 🟨 | 🟨 |

| 支持的能力 | 说明 |
| --- | --- |
| RealVideo 8 / RV30 | CV200产品配置选择real8语法及HAL；Linux不开放此格式，内部未完成路径不算支持。 |
| RealVideo 9 / RV40 | CV200产品配置选择real9语法及HAL；不能把备用hir9 stub当成正常生产路径。 |
| RealVideo 10独立变体覆盖 | 未发现独立RV10产品开关或明确的变体覆盖依据，不把所有RV40品牌变体自动算进来。 |

### VP6

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| VP6硬件解码 | ✅ | 🟨 |
| VP6F / VP6A独立变体 | 🟨 | 🟨 |

| 支持的能力 | 说明 |
| --- | --- |
| VP6硬件解码 | CV200配置实际选择vp6语法与vdm_hal_vp6；Linux不开放VP6。 |
| VP6F / VP6A独立变体 | 不从泛型枚举或SCD处理函数推断完整flip/alpha解码与输出合同。 |

### JPEG / MJPEG（JPGD）

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Baseline sequential，8-bit | ✅ | 🟨 |
| Extended sequential，8-bit（SOF1） | ✅ | 🟨 |
| 4:0:0 / 灰度 | ✅ | 🟨 |
| 4:2:0 | ✅ | 🟨 |
| 4:2:2水平采样 | ✅ | 🟨 |
| 4:2:2垂直采样 | ✅ | 🟨 |
| 4:4:4 | ✅ | 🟨 |
| Restart interval / DRI | ✅ | 🟨 |
| Progressive JPEG | ❌ | ❌ |
| Lossless / arithmetic JPEG | ❌ | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| Baseline sequential，8-bit | 独立JPGD解析器/HAL有完整路径；当前Linux没有JPGD节点，CPU JPEG不改变该列。 |
| Extended sequential，8-bit（SOF1） | 硬件parser同时接受SOF0/SOF1，但实际精度检查仅允许8-bit。 |
| 4:0:0 / 灰度 | BSP识别单分量并配置YUV400；CV200有对应JPGD采样路径。 |
| 4:2:0 | BSP解析采样因子并向JPGD写入对应格式；可输出半平面YUV。 |
| 4:2:2水平采样 | BSP区分YUV422_21并配置采样/输出几何。 |
| 4:2:2垂直采样 | BSP区分YUV422_12，不与水平采样混为一个布局。 |
| 4:4:4 | BSP有采样因子、几何和JPGD格式路径；不借用VDEC/JPGE节点。 |
| Restart interval / DRI | CV200配置保留DRI支持，硬件parser/设置路径可达。 |
| Progressive JPEG | BSP硬件资格检查拒绝progressive，软件libjpeg能力不算硬解。 |
| Lossless / arithmetic JPEG | 硬件parser拒绝对应SOF/DAC，精度限定8-bit。 |

### 未证实或未启用的硬件格式

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| DivX 3 | ❌ | ❌ |
| H.263独立硬解 | ❌ | ❌ |
| AVS2硬解 | ❌ | ❌ |
| AV1硬解 | ❌ | ❌ |
| PNG硬解 | ❌ | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| DivX 3 | CV200配置条目不足以证明实现；实际仅hidivx3/hi_hal_divx3空函数/stub。 |
| H.263独立硬解 | CV200生产配置未启用，SOS明确关闭；软件适配器不是VDH路径。 |
| AVS2硬解 | CV200没有被产品配置选择的AVS2语法/HAL；不能套用其他SoC实现。 |
| AV1硬解 | 本次CV200产品及VDH路径没有实现依据。 |
| PNG硬解 | libpng/TDE软件与显示链不等于专用JPGD/VDH解码器。 |

AVS/AVS+硬件路径依赖17,920字节的`hisilicon/histb-avsp.bin`。缺少固件、长度不符或DMA约束不满足时，请求必须在启动硬件前失败。独立场、profile/尺寸变更和极端工具组合的实机覆盖不能从源码状态外推。

## 编码

CV200生产VENC只启用H.264与JPGE。H.264产品配置的尺寸范围为176..1920×144..1088、picture尺寸4对齐、帧率1..60；这些是产品边界，不是所有组合的实测吞吐承诺。standalone JPGE的BSP校验范围为16..4096，两条编码器不能共用能力结论。

以下profile行表示已实现的编码配置/语法路径。Baseline/Main/High已分别完成320×240、25 fps的最小硬件编码输出及解码检查，不代表全部尺寸、速率与工具组合均已验收；解码输入使用CABAC/CAVLC也不能代替编码profile验证。

### H.264 / AVC

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Baseline：8-bit 4:2:0 | ✅ | ✅ |
| Main：8-bit 4:2:0 | ✅ | ✅ |
| High：8-bit 4:2:0 | ✅ | ✅ |
| I / IDR图像 | ✅ | ✅ |
| P图像 | ✅ | ✅ |
| CAVLC | ✅ | ✅ |
| CABAC | ✅ | ✅ |
| 多slice编码 / slice split | ✅ | 🟨 |
| 全局码率控制 | ✅ | ✅ |
| QP控制 | ✅ | ✅ |
| GOP控制 | ✅ | ✅ |
| 帧率控制 | ✅ | ✅ |
| 编码尺寸设置 | ✅ | ✅ |
| VUI sample aspect ratio | ❌ | ✅ |
| NV12：8-bit 4:2:0 UV | ✅ | ✅ |
| 独立Y/UV地址输入 | ✅ | ✅ |
| NV21：8-bit 4:2:0 VU | ✅ | 🟨 |
| Planar 4:2:0输入 | 🟨 | 🟨 |
| Semiplanar 4:2:2输入 | 🟨 | 🟨 |
| YUYV输入 | 🟨 | 🟨 |
| UYVY输入 | 🟨 | 🟨 |
| YVYU输入 | 🟨 | 🟨 |
| ROI区域QP | 🟨 | 🟨 |
| OSD叠加 | 🟨 | 🟨 |
| B帧编码 | ❌ | ❌ |
| Extended Profile | ❌ | ❌ |
| Constrained Baseline独立配置 | 🟨 | ❌ |
| 独立区域码率控制 | ❌ | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| Baseline：8-bit 4:2:0 | BSP接受profile 66；Linux实现对应SPS/PPS，使用CAVLC。 |
| Main：8-bit 4:2:0 | BSP接受profile 77；Linux有Main+CABAC配置路径，不暗示B帧编码。 |
| High：8-bit 4:2:0 | BSP接受profile 100；Linux默认使用High+CABAC，PPS关闭8×8 transform并使用默认矩阵，不代表启用全部High工具，也不承诺与BSP字节级相同。 |
| I / IDR图像 | 强制关键帧、GOP起点和SPS/PPS输出有实际路径。 |
| P图像 | 使用已重建的参考表面；当前VENC为I/P链路。 |
| CAVLC | Baseline的实际编码熵模式；不是解码器的CAVLC能力证明。 |
| CABAC | Main/High的PPS entropy flag和VEDU熵编码位均实现。 |
| 多slice编码 / slice split | BSP公共属性校验、channel参数与HAL的SliceSplit/SliceSize写入形成闭环，公开byte模式范围512..65535；不承诺运行中任意切换。Linux未接入，当前每帧清零对应PICFG1寄存器。JPGE拒绝slice split不能套用到H.264。 |
| 全局码率控制 | BSP有通道RC和bitrate属性；Linux为CBR-like帧级反馈，不宣称严格CBR或全套BSP RC策略。 |
| QP控制 | 全局I/P QP、min/max QP及固定QP路径；不是区域QP接口。 |
| GOP控制 | 控制I/P序列的关键帧间隔，不是帧率或B帧间隔。 |
| 帧率控制 | CV200产品范围1..60；Linux目前只接受整数帧率，30000/1001、60000/1001不在当前接口合同内；不代表所有尺寸/速率组合实测通过。 |
| 编码尺寸设置 | BSP有宽高属性和SPS crop语法；Linux raw队列宽16对齐、高2对齐，公开S_FMT及左上角crop selection，不提供任意位置或任意缩放。 |
| VUI sample aspect ratio | 所扫BSP自动SPS不写SAR；Linux的软件SPS生成及标准V4L2 SAR控件已实现。这是软件头处理差异，不是芯片物理能力差异。 |
| NV12：8-bit 4:2:0 UV | Linux公开NV12M多平面队列并支持DMA-BUF导入；pitch与可见宽度独立协商。 |
| 独立Y/UV地址输入 | BSP的SrcYAddr/SrcCAddr与HAL分别写入Y/C物理地址；Linux允许不同DMA-BUF fd及非零source data_offset。导入的压缩CAPTURE通过exporter的begin/end_cpu_access维护CPU访问，不能对no-map地址直接做cache操作。 |
| NV21：8-bit 4:2:0 VU | BSP实际源图像路径和VU selector有依据；Linux固定UV/NV12M。 |
| Planar 4:2:0输入 | BSP有Cr地址及专门stride分支，但尚未实机验证该布局；Linux未开放。 |
| Semiplanar 4:2:2输入 | BSP通过C stride×2抽取色度行，输出仍是4:2:0；不是High 4:2:2编码。 |
| YUYV输入 | BSP有packed 422地址/stride及独立selector；Linux未开放。 |
| UYVY输入 | BSP有packed 422布局路径；不能由该映射外推H.264 4:2:2输出。 |
| YVYU输入 | BSP有不同字节序selector；当前Linux仍仅NV12M。 |
| ROI区域QP | BSP有8区域结构/寄存器和使能路径，但缺公共setter及完整区域几何/QP写入闭环；Linux每帧关闭ROI。 |
| OSD叠加 | BSP有8层结构/寄存器和使能路径，但未找到公开配置及完整地址/stride/alpha写入闭环；Linux每帧关闭OSD。 |
| B帧编码 | CV200生产slice模型只有I/P，缺B slice、L1参考和编码重排合同。 |
| Extended Profile | BSP CheckPrivateAttr明确拒绝；Linux profile菜单也排除Extended。 |
| Constrained Baseline独立配置 | BSP有profile 66路径，但未证明独立约束配置；Linux profile菜单明确屏蔽Constrained Baseline，不能用FFmpeg的通用枚举代替驱动支持。 |
| 独立区域码率控制 | ROI中的区域QP不是每区域bitrate；没有相应公共RC接口。 |

### JPEG / MJPEG（JPGE）

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Baseline sequential，8-bit | ✅ | ✅ |
| 4:2:0输出 | ✅ | ✅ |
| 4:2:2输出 | ✅ | ✅ |
| 4:4:4输出 | ✅ | ✅ |
| NV12输入 | ✅ | ✅ |
| NV16输入 | ✅ | ✅ |
| NV24输入 | ✅ | ✅ |
| YUYV输入 | ✅ | ✅ |
| YVYU输入 | ✅ | ✅ |
| UYVY输入 | ✅ | ✅ |
| Planar YUV输入 | ✅ | 🟨 |
| Quality / 量化表 | ✅ | ✅ |
| 旋转 | 🟨 | 🟨 |
| Slice split | ❌ | ❌ |
| Progressive JPEG编码 | ❌ | ❌ |
| Lossless JPEG编码 | ❌ | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| Baseline sequential，8-bit | 独立JPGE输出SOF0/JFIF；每次完成一张图像，连续输出可作为MJPEG。 |
| 4:2:0输出 | 对应SOF采样值0x22；Linux默认NV12输入路径。 |
| 4:2:2输出 | 对应SOF采样值0x21；当前Linux已实现NV16和packed 422。 |
| 4:4:4输出 | 对应SOF采样值0x11；Linux有NV24输入与采样消息。 |
| NV12输入 | semiplanar 420，有独立地址、stride和输出边界检查。 |
| NV16输入 | semiplanar 422；与H.264 VENC支持范围分开。 |
| NV24输入 | semiplanar 444；不表示H.264支持4:4:4编码。 |
| YUYV输入 | JPGE packed 422及对应字节序selector。 |
| YVYU输入 | JPGE packed 422及对应字节序selector。 |
| UYVY输入 | JPGE packed 422及对应字节序selector。 |
| Planar YUV输入 | BSP standalone JPGE有Y/C/V地址与planar路径；当前Linux仅semi-planar/packed输入。 |
| Quality / 量化表 | BSP Qlevel=1..99；Linux JPEG quality=1..100，使用自身量化表规则。 |
| 旋转 | standalone JPGE有rotation寄存器、stride和JFIF尺寸调整；VENC附带JPEG属性拒绝非0旋转。两条入口不能混为一谈，Linux未开放。 |
| Slice split | standalone JPGE资格检查拒绝SlcSplitEn!=0；寄存器字段不构成公开支持。 |
| Progressive JPEG编码 | 没有相应SOF2和多scan编码路径，不能以软件JPEG能力代替。 |
| Lossless JPEG编码 | 公开JPGE为8-bit baseline DCT编码，不提供lossless接口。 |

### 其他编码格式

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| HEVC / H.265编码 | ❌ | ❌ |
| AVS / AVS+编码 | ❌ | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| HEVC / H.265编码 | CV200 venc_make.cfg只启用H.264和JPGE，不能套用MV200的HEVC VENC。 |
| AVS / AVS+编码 | CV200没有对应VENC生产配置及硬件消息合同；AVS+转码输出是H.264。 |

VENC格式转换表还出现planar 422/444、NV42、400/411/410等名称，但实际H.264源图像路径没有证明这些布局的完整转换，部分采样类型为VENC_YUV_NONE；不列为已知可用输入，也不承诺monochrome、High 10、High 4:2:2或High 4:4:4输出。

## 后处理

后处理沿用同一组“已做／未做／不支持”状态；软件滤镜单独标明，不计作硬件能力。

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | VDEC tile-to-linear | 当前 `histb-vpss` 在VDEC内部完成tile到线性帧转换，不提供独立通用节点 |
| ✅ | 内部ZME缩放核心 | 装载8-tap luma / 4-tap chroma多档系数并按比例选择；已有3840×2160 Main10到1920×1080 NV12输出记录，50 fps元数据本身不证明整链持续实时吞吐 |
| ✅ | 解码输出尺寸协商 | capture只向下协商，宽4像素、高2行对齐，缩小比严格低于16:1；可同时完成Main10到8-bit NV12转换，不提供公开放大 |
| ✅ | VPSS field-rate DEI | FFmpeg显示重排后，`hivxedei`按LAST/CUR/NEXT1/NEXT2提交四场DEI并将独立DMA-BUF交给VENC；当前公共入口限本VDEC的NV12表面、最大1920×1088，不接受任意导入buffer或P010 |
| ✅ | 内置PQ参数与条件DBM | 已装载含DEI/FMD、DB/DM/DR/DS、SNR/TNR区域的默认表及部分FHD覆盖；四场节点按宽>128、场高>64启用DBM。参数已存在不等于每个处理块均开启 |
| ✅ | CPU bob（软件） | extra提供 `hivxebob`，不是VPSS硬件DEI；`bwdif`未编入当前交付FFmpeg，不能直接套用其命令 |
| 🟨 | 完整MCDI去隔行 | BSP有RGME、块/区域运动、投影缓冲及历史写回链；Linux未完成对应生命周期，当前清除MCDI/MEDS使能位 |
| 🟨 | TNR/SNR/MCNR | BSP有时域/空域降噪及统计链；Linux虽有部分默认参数，仍清除TNR/SNR使能位，未提供完整处理链和用户调节接口 |
| 🟨 | 独立DB/DM、dering、deshoot调节 | 不同于上面的内置参数和条件DBM，Linux尚未提供独立公共PQ调节接口，不承诺全部增强效果已验收 |
| 🟨 | 旋转与多输出端口 | BSP有旋转和多port调度；Linux尚未开放，当前后处理主要服务VDEC内部路径 |
| 🟨 | 完整HDR tone mapping / 3D链 | 当前未接通色彩元数据、tone mapping、双视角缓冲及显示链闭环；Main10转NV12不等于HDR转SDR，MVC parser也不等于完整3D输出 |

所以“解码器能输出隔行内容”“CPU 滤镜能 bob”和“独立 MCDEI 硬件已开放”是三件不同的事。

当前DEI是VPSS四场硬件处理，不等于原厂完整MCDI、运动估计、TNR或PQ链。旧`dei`/`dei_field_rate`模块参数已移除；DEI由显示重排后的`hivxedei`显式提交，不改变其他解码会话的capture布局。

非安全 Linux DEI 节点使用 CV200 `CTRL.prot=2`，并为 IGBM/IFMD 提供真实的 32 KiB 统计写回 DMA 区；DBM 按原厂四场节点的尺寸条件启用。统计写回地址不能留为 0。补齐这些配置后，1006 的 TVH Spawn 已实测输出 1920×1080 逐行 50 fps（45 秒抓取含 2170 张视频帧、43.4 秒视频，输出解码无错误）；这不代替 UHD 输入、ALSA 时钟或完整 MCDI 的独立验收。

当前 `hivxe-transcode` 使用同一套尺寸/帧率策略：SD/HD保留原尺寸；逐行25/50保留帧率，25帧/秒隔行输入按场输出50p；超过1920×1080的逐行输入由VPSS按比例缩小。码率/GOP不由TVH profile名称推断。picture cadence优先于开流时的packet/r_frame_rate猜测。

AVS+ MPEG-TS必须编入`cavsvideo` parser以合并跨PES的两场。VENC的压缩包使用独立CPU内存，立即回收有限的CAPTURE DMA缓冲，防止音频交错封装把编码队列全部占住；像素表面仍由DMA-BUF交接。这两类CPU复制不能混为一谈。

## 错误处理

硬件完成失败必须返回错误，不能用 repeat-last、插值或强制交付坏帧掩盖；
这份文档只记录公开接口和已确认的能力，不把寄存器字段、stub 或软件
fallback 当成硬件支持。

## Tvheadend 使用 HiVXE 编解码

使用顺序是：**安装支持HiVXE的内核/模块和固件 → 安装extra编解码程序及转码脚本 → 在Tvheadend新增Stream Profile → 播放时选择该profile**。不是只创建一个profile就能让普通内核或系统FFmpeg获得海思硬件能力。

这里有两种不同的profile：TVH的 **Stream Profile** 决定调用哪个转码程序；H.264的 **Baseline/Main/High profile** 决定输出码流语法。给TVH profile取名 `hivxe-720p` 不会自动修改分辨率或H.264 profile。

### 1. 安装前提

| 层次 | 必需内容 | 说明 |
| --- | --- | --- |
| 内核/设备树 | 本项目HiVXE VDEC、VENC、VPSS及media SMMU支持 | 使用匹配板卡的[内核/固件包](https://github.com/HiSilicon-Development/firmware/releases)，模块须匹配运行内核；更换内核后需重启。extra不会安装内核 |
| 解码固件 | `hisilicon/histb-h264-cabac.bin`、`histb-hevc-cabac.bin`；AVS/AVS+另需 `histb-avsp.bin` | 安装到内核固件搜索路径中的 `hisilicon/`，通常为 `/lib/firmware/hisilicon/`；AVSP必须为17,920字节 |
| 用户态编解码 | `hivxe-ffmpeg`、`hivxe-ffprobe`及匹配的私有库 | 使用[extra安装包](https://github.com/HiSilicon-Development/extra)，不能用发行版普通FFmpeg替代本项目Request/VPSS扩展 |
| 转码入口 | `/usr/local/bin/hivxe-transcode` | extra提供的可执行脚本，接收stdin MPEG-TS并在stdout输出MPEG-TS；它不是自动创建的TVH配置 |
| Tvheadend | MPEG-TS Spawn类型的Stream Profile及设备访问权限 | TVH服务用户需能执行脚本、访问相应media/video节点；按系统video组或udev规则授权，组权限变更后重启服务 |

参考defconfig将 `CONFIG_VIDEO_HISTB_VDEC`、`CONFIG_VIDEO_HISTB_VENC`、`CONFIG_VIDEO_HISTB_JPGE` 配为模块；VDEC选择VPSS和共用SMMU。节点注册成功、固件齐全与FFmpeg组件已编入是不同条件，不能只看Kconfig或磁盘上有 `.ko` 文件。

在Debian 12 arm64、运行systemd的设备上，以root按extra的[安装说明](https://github.com/HiSilicon-Development/extra#readme)执行：

```sh
curl -fL https://raw.githubusercontent.com/HiSilicon-Development/extra/main/install.sh -o install.sh
bash install.sh
```

**该安装器还会安装OSCam，并覆盖其程序、随包默认配置和service；检测到旧安装时会询问是否先备份。HiVXE程序和私有库直接覆盖更新。** 如需保留现有配置，先阅读安装说明再执行，不要把它误认为仅安装一个TVH profile。

安装后可查看交付组件：

```sh
/usr/local/bin/hivxe-ffmpeg -hide_banner -hwaccels
/usr/local/bin/hivxe-ffmpeg -hide_banner -encoders
/usr/local/bin/hivxe-ffmpeg -hide_banner -filters
```

这些列表用于确认组件已编入，不代替硬件实际运行。当前extra的输入硬解选择如下；内核能力矩阵不等于该精简安装包全部启用：

| 输入视频 | Linux接口 | 当前extra的 `hivxe-transcode` |
| --- | --- | --- |
| H.264、HEVC、MPEG-2、AVS/AVS+ | V4L2 Request | 已编入对应decoder/hwaccel，由输入码流选择；AVS+需 `cavsvideo` parser合并两场 |
| VP8、VP9 | V4L2 Request | 内核已做，当前包未编入相应解码链 |
| MPEG-4 Part 2、VC-1 | Stateful V4L2 M2M | 内核已有相应路径，当前包未启用对应decoder；固定Request脚本也不会自动改走stateful接口 |
| MPEG-1、RV、VP6、JPGD、完整MVC | 尚无对应公开Linux路径 | 不能添加一个FFmpeg flag就启用 |

### 2. 新建 Stream Profile

打开 `Configuration → Stream → Stream Profiles`，新增 **MPEG-TS Spawn**（部分界面显示 `MPEG-TS Spawn/built-in`）：

| 字段 | 设置 |
| --- | --- |
| Name | 例如 `hivxe`，仅用于识别 |
| Enabled | 启用 |
| Command line | `/usr/local/bin/hivxe-transcode` |
| Mime type | `video/mp2t` |

保存后，在客户端/播放请求中选择这个profile，并确保相应用户有使用权限。继续使用 `pass` 等其他profile不会调用它。此方式直接启动外部编解码程序，不要求给TVH内置转码器另加海思私有调用代码；extra安装器也不会代替用户创建TVH Stream Profile。

**Command line只填脚本绝对路径**，不需要粘贴下面的FFmpeg长命令，也不需要 `<`、`>`、管道或环境变量。TVH把选中服务的TS送入stdin，脚本把转码TS写回stdout，stderr用于日志。

默认脚本固定使用 `/dev/media0`。已经正常工作的设备无需改动；若设备注册顺序不同，应在自定义脚本中改为实际VDEC节点，而不是DVB adapter编号。VDEC的media model为 `HiSilicon Hi3798CV200 VDH`。可选装 `v4l-utils` 后用 `media-ctl -d /dev/media0 -p` 查看；需要遍历时在终端执行：

```sh
for node in /dev/media*; do
    [ -e "$node" ] || continue
    media-ctl -d "$node" -p
done
```

`media-ctl`是诊断工具，不是extra安装器提供的程序，也不是每次转码前必须运行的步骤。

### 3. 默认脚本及处理策略

extra的 [`hivxe-transcode`](https://github.com/HiSilicon-Development/extra/blob/main/payload/bin/hivxe-transcode) 当前内容为：

```sh
#!/bin/sh
export HISTB_V4L2REQUEST_DEFER_CAPTURE_WAIT=auto
export HISTB_V4L2REQUEST_MAIN10_NV12=1
export HISTB_V4L2REQUEST_MAX_CAPTURE_SIZE=1920x1080
exec /usr/local/bin/hivxe-ffmpeg -hide_banner -loglevel warning -nostdin \
  -init_hw_device v4l2request=hivxe:/dev/media0 \
  -hwaccel v4l2request -hwaccel_device hivxe -hwaccel_output_format drm_prime \
  -threads 1 -extra_hw_frames 0 \
  -f mpegts -probesize 2097152 -analyzeduration 5000000 -fflags +genpts+discardcorrupt \
  -i pipe:0 -map 0:v:0 -map 0:a? -map 0:s? \
  -vf hivxedei \
  -c:v h264_v4l2m2m -num_capture_buffers 16 -b:v 8000000 -g 50 \
  -c:a copy -c:s copy \
  -fps_mode passthrough -muxdelay 0 -muxpreload 0 -flush_packets 1 -mpegts_flags +resend_headers \
  -f mpegts pipe:1
```

默认是 **Request硬解 → VPSS输出NV12/按需缩小 → 按需硬件去隔行 → H.264硬编码 → TS输出**，视频目标8 Mbps、GOP 50、驱动默认High profile，音频/字幕复制：

| 输入情况 | 默认处理 |
| --- | --- |
| SD/HD逐行25p或50p | 保留源尺寸与帧率，不强制放大到1080，不额外翻倍 |
| 1080i50，即每秒25张隔行帧/50场 | `hivxedei`按显示顺序调用VPSS四场DEI，输出1080p50 |
| 超过1920×1080的逐行视频 | VPSS在capture阶段按比例缩小到上限内；50p输入保留50p时间戳，是否持续实时须看实际吞吐 |
| HEVC Main10 | 先按10-bit重建，再由VPSS转8-bit NV12供VENC使用；不是HDR→SDR tone mapping |

`hivxedei`要求Request/NV12 DRM PRIME硬件帧。普通FFmpeg解码层虽可能回退软件格式，**这个固定脚本没有完整的软件自动回退链**；不能在保留硬件滤镜的同时指望CPU帧继续输出。若源把交错画面误标为逐行，当前自动DEI会透传，不能靠 `-g` 或输出 `-r` 修复场标志。

### 4. flag 是什么，现有参数做什么

这里把命令行选项统称为 **flag**：例如 `-hwaccel v4l2request` 选择硬解，`-b:v 8000000` 设置视频目标码率。三个 `HISTB_...` 则是本项目FFmpeg扩展读取的**环境变量**，不是标准FFmpeg命令行flag，也不是内核模块参数。

输入选项放在 `-i pipe:0` 前；编码/输出选项放在其后。下面解释的是脚本内实际传给FFmpeg的参数，不是可追加给脚本的参数接口。

#### 环境变量

| 设置 | 启用的行为 | 边界 |
| --- | --- | --- |
| `HISTB_V4L2REQUEST_DEFER_CAPTURE_WAIT=auto` | 逐行且capture为NV12时延后到消费者等待硬件完成，实现流水化 | 包括原尺寸逐行NV12，不只下采样；隔行或非NV12仍先等待。错误检查移到消费前，不是跳过错误 |
| `HISTB_V4L2REQUEST_MAIN10_NV12=1` | Main10优先选择NV12 capture，走VPSS的10→8-bit转换 | 为8-bit VENC准备输入；不能启用H.264 High10编码或HDR tone mapping |
| `HISTB_V4L2REQUEST_MAX_CAPTURE_SIZE=1920x1080` | 超限时按可见宽高比向下缩小，小源保留coded尺寸 | 输出宽4对齐、高2对齐，驱动缩小比须严格低于16:1；不缩小VDH重建面，也不提供放大 |
| `HISTB_V4L2REQUEST_CAPTURE_SIZE=1280x720`（默认未设置） | 显式请求固定capture尺寸 | 优先于MAX，不能超过coded尺寸，也不自动保持宽高比；不适合混合SD/HD频道的统一入口 |

前两个变量按“是否存在”判断，设为 `0` **不是关闭**；DEFER只有精确值 `auto` 才应用上面的条件，其他存在的值会无条件请求延后等待。关闭应在脚本中去掉相应export并 `unset` 该变量。去掉MAIN10_NV12后Main10要求P010，不能继续沿用只接受NV12的DEI/H.264编码链。

前三个export覆盖调用者同名设置；但脚本没有清除继承的CAPTURE_SIZE，所以它仍可能覆盖MAX策略。需要统一按上限缩小时，应确保自定义脚本中 `unset HISTB_V4L2REQUEST_CAPTURE_SIZE`。

#### FFmpeg 参数

| 参数 | 作用及限制 |
| --- | --- |
| `exec` | shell被FFmpeg替换，TVH管理的是转码进程；不是FFmpeg选项 |
| `-hide_banner -loglevel warning` | 隐藏banner，只保留warning及更严重日志 |
| `-nostdin` | 关闭键盘交互，不妨碍 `pipe:0` 读取媒体数据 |
| `-init_hw_device v4l2request=hivxe:/dev/media0` | 创建名为hivxe的Request设备，节点固定media0，不自动寻找VDEC |
| `-hwaccel v4l2request -hwaccel_device hivxe` | 为输入视频选择Request硬解并复用该设备，不等于选择stateful M2M decoder |
| `-hwaccel_output_format drm_prime` | 保留DRM PRIME硬件帧，供VPSS与VENC的DMA-BUF路径使用 |
| `-threads 1 -extra_hw_frames 0` | 请求一个解码线程、用户额外硬件帧增量为0；参考池和FFmpeg调度仍可能另加帧，不是总共只分配零个额外buffer |
| 输入侧 `-f mpegts` | 强制输入为MPEG-TS |
| `-probesize 2097152 -analyzeduration 5000000` | 探测预算2 MiB、分析时间预算5秒；不是总启动延迟保证 |
| `-fflags +genpts+discardcorrupt` | 补生成PTS、丢弃已标记损坏的packet，不修复所有码流错误 |
| `-i pipe:0` | 从stdin读取TVH送来的TS |
| `-map 0:v:0` | 必须选择第一个视频；不复制其他视频和全部节目结构 |
| `-map 0:a? -map 0:s?` | 选择所有音频/字幕，末尾 `?` 允许相应流不存在 |
| `-vf hivxedei` | 逐行透传、隔行按场输出；自动硬件DEI已启用，不需再加旧的dei模块参数 |
| `-c:v h264_v4l2m2m` | 选择VENC硬编码H.264；未指定profile时使用当前驱动默认High+CABAC，仍只有I/P链 |
| `-num_capture_buffers 16` | 为编码器的压缩码流CAPTURE队列申请16个buffer，实际数量以驱动返回为准；不是解码DPB或原始图像池 |
| `-b:v 8000000` | 视频目标8 Mbps，当前为CBR-like帧级反馈，不是严格HRD-CBR或恒定TS muxrate |
| `-g 50` | 关键帧间隔50张输出图像；50p时约1秒、25p时约2秒，不是输出帧率 |
| `-c:a copy -c:s copy` | 复制音频/字幕packet，不做AAC转码或字幕格式转换；输入字幕仍须适合TS输出 |
| `-fps_mode passthrough` | 不为固定帧率主动重复/丢弃滤镜输出，不强制50 fps，也不保证实时 |
| `-muxdelay 0 -muxpreload 0 -flush_packets 1` | 降低mux阶段缓存并逐packet刷新，不能消除探测、解码重排、DEI和播放器延迟 |
| `-mpegts_flags +resend_headers` | 重发PAT/PMT表，不是H.264 SPS/PPS的同义词 |
| 输出侧 `-f mpegts pipe:1` | 将MPEG-TS写到stdout；不要在脚本中向stdout打印诊断文字 |

### 5. 可选特性与自定义 profile

**`hivxe-transcode`没有 `$@` 参数转发。** 在TVH中写 `hivxe-transcode -b:v 6000000` 不会修改码率；给它外加MAX环境变量也会被脚本内export覆盖。需要不同配置时，复制一份脚本，修改其中的export/FFmpeg参数，再让另一个TVH Stream Profile调用这个副本；不必修改原厂BSP或重新实现TVH集成。

| 想要的特性 | 在自定义脚本中修改 | 限制 |
| --- | --- | --- |
| 最高720p、保留小源 | 将MAX的export改为 `HISTB_V4L2REQUEST_MAX_CAPTURE_SIZE=1280x720` | 保持比例、不放大；先清除CAPTURE_SIZE覆盖 |
| 固定capture尺寸 | 另设 `HISTB_V4L2REQUEST_CAPTURE_SIZE=1280x720` | 只用于源coded尺寸足够大的固定输入，可能改变宽高比；不是通用720p上限 |
| Baseline / CAVLC输出 | 在输出编码参数中加入 `-profile:v 66` | 当前FFmpeg使用数值ID；不是Constrained Baseline独立配置 |
| Main / CABAC输出 | 加入 `-profile:v 77` | 仍只有I/P，不会启用B帧 |
| High / CABAC输出 | 加入 `-profile:v 100`，或保留默认 | 当前High不启用8×8 transform，不是全部High工具开关 |
| 更低/更高目标码率 | 替换现有 `-b:v`，例如 `-b:v 6000000` | 只控制视频目标；CBR-like，不保证每秒字节数恒定 |
| 调整关键帧间隔 | 替换为 `-g 25` 或 `-g 50` | 单位是输出图像，不改变分辨率/帧率 |
| 限制量化范围 | 加入 `-qmin 20 -qmax 40` | H.264范围0..51且min≤max；不是固定QP，也不是ROI区域QP |
| 不输出字幕 | 删除 `-map 0:s?` 和 `-c:s copy` | 仅去掉字幕，不改变视频硬件链 |
| 调试硬件选择/吞吐 | 将日志改为 `-loglevel info`，需要时使用 `debug`，另开终端运行 `hivxe-top` | 日志走stderr，不能混入TS stdout；不要在正常长驻流中使用 `-xerror` |

例如创建一个最高720p、Main、6 Mbps的副本，以root执行一次：

```sh
cp /usr/local/bin/hivxe-transcode /usr/local/bin/hivxe-transcode-720p
chmod 0755 /usr/local/bin/hivxe-transcode-720p
```

编辑该副本：在三个export前加入 `unset HISTB_V4L2REQUEST_CAPTURE_SIZE`，把MAX改为1280x720，并把原编码参数行**替换**为：

```text
  -c:v h264_v4l2m2m -profile:v 77 -num_capture_buffers 16 -b:v 6000000 -g 50 \
```

其他行保持不变。在TVH再建一个名为 `hivxe-720p` 的MPEG-TS Spawn profile，其Command line填 `/usr/local/bin/hivxe-transcode-720p`。今后更新extra会覆盖标准入口，副本需自行同步上游脚本变化。

以下功能**没有一个可直接启用的flag**：完整MVC/JPGD/RV/VP6/MPEG-1路径、VENC slice split/ROI/OSD、完整MCDI/TNR/SNR、H.264 B帧编码、High10/4:2:2/4:4:4编码及HDR tone mapping。FFmpeg通用选项或枚举的存在不能补齐驱动实现；不要用 `-bf`、`-profile:v 110` 等选项把它们写成可用。

如需CPU软件解码，应另建脚本并使用相应软件decoder和软件滤镜，不能把CPU帧直接交给 `hivxedei`。当前包提供 `hivxebob`，没有 `bwdif`、`hwdownload` 或 `setfield` 滤镜；强制处理误标为progressive的交错源需要先补齐相应组件/处理链，不能照抄未编入的滤镜命令。

音频默认保留 `-c:a copy`。当前包虽有AAC encoder，但未编入 `aresample`：例如MP2解码的s16p不能自动转成AAC所需的fltp，实测直接改为 `-c:a aac -b:a 128000` 会失败，不能把它写成通用AAC转码开关。需要此类音频转码时，须先补齐带采样格式转换能力的匹配FFmpeg；该处理在CPU执行，不是海思硬件音频编码。

### 6. 资源与运行边界

- Main10降为NV12不减少VDH的10-bit重建内存；即使capture为1080p，4K HEVC仍按4K分配重建面。当前内核将UHD HEVC解码capture队列限制为8个buffer；它与编码器的 `-num_capture_buffers 16` 是两个不同队列。
- 解码完成状态、slice范围及SMMU错误仍由内核校验；不能用repeat-last或交付坏帧掩盖故障。组播开流可能从GOP中间开始，等待有效参数集/IDR后应持续输出，持续坏帧不能当成正常启动现象。
- 1080i50→1080p50已有TVH Spawn运行记录；4K50等组合仍需区分输出帧率元数据、实际帧数和持续实时吞吐。没有核验过的尺寸/profile/速率组合不作实时承诺。
- Spawn字段不是shell，不会展开 `$MEDIA_NODE`、重定向、管道或 `&&`。多行shell与环境设置应放在可执行脚本里；Tvheadend 4.3的直接命令解析最多保留63个参数，调用脚本也避免长命令被截断。
- 保留可选音频/字幕映射的 `?`；不要把GOP、编码buffer数、H.264 profile或TVH profile名称当作帧率、解码池大小或能力启用证明。

## 公开来源

本页以CV200产品配置、可达parser/HAL及当前Linux接口为依据；VFMW厂商汇编由REA/Ghidra辅助核对，并与C实现、配置选择和寄存器消息交叉验证。宏、头文件枚举、旧构建对象或告警字符串单独存在均不构成完整支持证据。

- **CV200 VDEC 产品配置（主依据 SPC060）**：
  [SPC060 `vfmw_config.cfg`](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/vfmw_config.cfg)、
  [SPC060 `vfmw_make.cfg`](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/vfmw_make.cfg)。
- **SPC050 交叉参考**：
  [SPC050 `vfmw_config.cfg`](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/vfmw_config.cfg)、
  [SPC050 `vfmw_make.cfg`](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/vfmw_make.cfg)。
- **CV200 VENC 产品配置**：
  [SPC060 `venc_make.cfg`](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/venc/venc_v1.0/firmware/product/Hi3798CV200/32Bit/venc_make.cfg)、
  [SPC050 `venc_make.cfg`](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC050/blob/master/source/msp/drv/venc/venc_v1.0/firmware/product/Hi3798CV200/32Bit/venc_make.cfg)。
- **Linux 当前接口入口**：
  [VDEC](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/histb-vdec-core.c)、
  [VENC](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/histb-venc-core.c)、
  [JPGE](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/histb-jpge.c)。
- **BSP语法与HAL交叉核对**：
  [H.264语法](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/32bit/core/syntax/h264.S)、
  [MVC语法](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/32bit/core/syntax/mvc.S)、
  [VP9 HAL](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/32bit/core/driver/HiVDHV4R3C1/vdm_hal_vp9.S)、
  [VENC属性校验](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/venc/venc_v1.0/drv_venc.c)、
  [VENC HAL](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/venc/venc_v1.0/hal_venc.c)。
- **Linux构建与后处理**：
  [HiVXE Makefile](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/Makefile)、
  [VPSS](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/histb-vpss.c)。
