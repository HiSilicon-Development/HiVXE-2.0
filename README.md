# HiSilicon Hi3798CV200 HiVXE 编解码能力

本文先从Hi3798CV200 BSP的产品配置、解析器、HAL和实际调用链确定硬件能力，再与本仓库Linux 6.12.y驱动及FFmpeg接口逐项对照。解码与编码分开，profile、位深、色度、帧类型和工具分别列行。

`Linux 6.12.y`指本项目带HiVXE驱动补丁的内核，不是原生上游6.12内核。能力矩阵描述源码和公开硬件接口；具体安装包可能裁剪FFmpeg组件，不能把CPU软件解码或头文件枚举当成硬件支持。

## 状态

| 状态 | BSP列 | Linux 6.12.y列 |
| --- | --- | --- |
| ✅ | 匹配CV200的产品配置及可达解析器/HAL或调用链有实现依据 | 本仓库已实现并开放对应硬件接口；不代表所有样片、profile组合均实机验收 |
| 🟨 | 有内部结构、部分HAL或条件配置，但不能证明完整可用路径 | 有部分实现，或路径尚未公开/验证，不能作为完整支持承诺 |
| ❌ | 明确拒绝、未启用、仅stub，或没有足够CV200实现证据 | 未开放该硬件能力，或当前路径明确拒绝 |

两列独立判断。`BSP ✅ / Linux ❌`不表示Linux有隐藏开关；`BSP 🟨`也不表示芯片物理上绝不可能实现。表后的说明记录接口、输出布局及未验证边界。CPU软件路径只在说明中标出，不改变硬件列状态。

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
| CPU 软件路径 | FFmpeg 软件解码器，例如 `cavs` | 不依赖 HiVXE 硬件节点，作为默认路径和硬件协商失败时的 fallback |
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

不支持High 10、High 4:2:2、High 4:4:4、FMO、redundant picture、data partition、SP/SI slice和完整SVC分层解码；CV200 SPS还拒绝transform bypass。MVC另列，不从profile ID或parser helper推断Linux双视角能力。

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
| Main Still Picture | PTL解析和8-bit IRAP/I路径存在，但未证明profile 3的独立完整覆盖。 |
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
| 8-bit 4:2:0 | ✅ | ❌ |
| I图像 | ✅ | ❌ |
| P图像 | ✅ | ❌ |
| B图像 | ✅ | ❌ |

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

不支持data partitioning、NEWPRED、reduced-resolution、scalability、OBMC和static sprite；GMC/S-VOP只对应动态sprite。

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
| Stereo High（profile 128）双视角 | ✅ | ❌ |
| Multiview High（profile 118）双视角 | ✅ | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| Stereo High（profile 128）双视角 | BSP subset-SPS、inter-view reference及H.264 HAL消息可达；Linux没有公开MVC前端/双视角输出。 |
| Multiview High（profile 118）双视角 | CV200 BSP最多两个视角；不承诺任意N视角。 |

CV200 subset-SPS明确限制最多两个视角；Linux中的MVC helper及接受118/128的profile ID不构成完整MVC硬件接口。

### RealVideo

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| RealVideo 8 / RV30 | ✅ | ❌ |
| RealVideo 9 / RV40 | ✅ | ❌ |
| RealVideo 10独立变体覆盖 | 🟨 | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| RealVideo 8 / RV30 | CV200产品配置选择real8语法及HAL；Linux不开放此格式，内部未完成路径不算支持。 |
| RealVideo 9 / RV40 | CV200产品配置选择real9语法及HAL；不能把备用hir9 stub当成正常生产路径。 |
| RealVideo 10独立变体覆盖 | 未发现独立RV10产品开关或明确的变体覆盖依据，不把所有RV40品牌变体自动算进来。 |

### VP6

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| VP6硬件解码 | ✅ | ❌ |
| VP6F / VP6A独立变体 | 🟨 | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| VP6硬件解码 | CV200配置实际选择vp6语法与vdm_hal_vp6；Linux不开放VP6。 |
| VP6F / VP6A独立变体 | 不从泛型枚举或SCD处理函数推断完整flip/alpha解码与输出合同。 |

### JPEG / MJPEG（JPGD）

| 支持的能力 | BSP | Linux 6.12.y |
| --- | --- | --- |
| Baseline sequential，8-bit | ✅ | ❌ |
| Extended sequential，8-bit（SOF1） | ✅ | ❌ |
| 4:0:0 / 灰度 | ✅ | ❌ |
| 4:2:0 | ✅ | ❌ |
| 4:2:2水平采样 | ✅ | ❌ |
| 4:2:2垂直采样 | ✅ | ❌ |
| 4:4:4 | ✅ | ❌ |
| Restart interval / DRI | ✅ | ❌ |
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

以下profile行表示已实现的编码配置/语法路径。单路默认路径已有运行证据，但尚未逐Baseline/Main/High完成板卡输出语法矩阵；不把解码输入使用CABAC/CAVLC的实验写成编码profile验收。

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
| 全局码率控制 | ✅ | ✅ |
| QP控制 | ✅ | ✅ |
| GOP控制 | ✅ | ✅ |
| 帧率控制 | ✅ | ✅ |
| 编码尺寸设置 | ✅ | ✅ |
| VUI sample aspect ratio | ❌ | ✅ |
| NV12：8-bit 4:2:0 UV | ✅ | ✅ |
| 独立Y/UV地址输入 | ✅ | ✅ |
| NV21：8-bit 4:2:0 VU | ✅ | ❌ |
| Planar 4:2:0输入 | 🟨 | ❌ |
| Semiplanar 4:2:2输入 | 🟨 | ❌ |
| YUYV输入 | 🟨 | ❌ |
| UYVY输入 | 🟨 | ❌ |
| YVYU输入 | 🟨 | ❌ |
| ROI区域QP | 🟨 | ❌ |
| OSD叠加 | 🟨 | ❌ |
| B帧编码 | ❌ | ❌ |
| Extended Profile | ❌ | ❌ |
| 独立区域码率控制 | ❌ | ❌ |

| 支持的能力 | 说明 |
| --- | --- |
| Baseline：8-bit 4:2:0 | BSP接受profile 66；Linux实现对应SPS/PPS，使用CAVLC。 |
| Main：8-bit 4:2:0 | BSP接受profile 77；Linux有Main+CABAC配置路径，不暗示B帧编码。 |
| High：8-bit 4:2:0 | BSP接受profile 100；当前Linux的High PPS使用默认矩阵，不能称与BSP字节级相同。 |
| I / IDR图像 | 强制关键帧、GOP起点和SPS/PPS输出有实际路径。 |
| P图像 | 使用已重建的参考表面；当前VENC为I/P链路。 |
| CAVLC | Baseline的实际编码熵模式；不是解码器的CAVLC能力证明。 |
| CABAC | Main/High的PPS entropy flag和VEDU熵编码位均实现。 |
| 全局码率控制 | BSP有通道RC和bitrate属性；Linux为CBR-like帧级反馈，不宣称严格CBR或全套BSP RC策略。 |
| QP控制 | 全局I/P QP、min/max QP及固定QP路径；不是区域QP接口。 |
| GOP控制 | 控制I/P序列的关键帧间隔，不是帧率或B帧间隔。 |
| 帧率控制 | CV200产品范围1..60；当前Linux接口校验请求值，不代表所有尺寸/速率组合实测通过。 |
| 编码尺寸设置 | BSP有宽高属性和SPS crop语法；Linux公开S_FMT及左上角crop selection，不提供任意位置或任意缩放。 |
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
| Planar YUV输入 | ✅ | ❌ |
| Quality / 量化表 | ✅ | ✅ |
| 旋转 | 🟨 | ❌ |
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

后处理与codec能力分开。下表沿用单列成熟度：`🟦`仅表示BSP有对应处理链而Linux未开放；不改变前述编解码矩阵两列的独立含义。

| 状态 | 能力 | 说明 |
| --- | --- | --- |
| ✅ | VDEC tile-to-linear | 当前 `histb-vpss` 在 VDEC 内部完成 tile 数据面到线性帧的转换，不提供独立通用节点 |
| ✅ | 内部 ZME 缩放核心 | `histb-vpss` 装载 8-tap luma / 4-tap chroma 多档系数，并按水平、垂直比例选择滤波器；3840x2160 Main10 50 fps 到 1920x1080 NV12 50 fps 已完成连续实机验证 |
| ✅ | 解码输出尺寸协商 | VDEC capture 允许从 coded size 向下协商，水平按 4 像素、垂直按 2 行对齐，缩小比严格低于 16:1；VPSS 同时完成 Main10 到 8-bit NV12 转换，不提供公开放大承诺 |
| ✅ | VPSS field-rate DEI | VDEC先交付request自身的woven NV12；FFmpeg显示重排后，`hivxedei`按LAST/CUR/NEXT1/NEXT2调用VPSS并以独立DMA-BUF交给VENC。TFF/BFF带B帧移动标记已验证无回退，不能再在解码完成顺序中维护四场历史 |
| 🟨 | CPU bob/反交错 | `hivxebob`/`bwdif` 是不依赖 VPSS DEI 的 CPU fallback；可用，但不是当前 1080i50 硬件转码的首选路径 |
| 🟦 | 完整 MCDI 去隔行 | BSP 有 RGME、块/区域运动、投影缓冲和历史帧写回链；当前 Linux 尚未建立可验证的完整生命周期 |
| 🟦 | TNR/SNR/MCNR | BSP 有时域/空域降噪寄存器、统计 buffer 及与 MCDEI 的连接；当前 Linux 未移植算法参数和用户接口 |
| 🟦 | DB/DM、dering、deshoot | BSP 有去块、细节增强、振铃和过冲控制参数；它们与 VPSS 节点及 PQ 表绑定，当前 Linux 未公开 |
| 🟦 | 旋转与多输出端口 | 原厂 VPSS 有旋转输出和多 port 调度；当前 Linux 只把后处理用于 VDEC 内部路径 |
| ❌ | HDR / 3D | 需要配套色彩元数据、tone mapping、双视角缓冲和显示链约定；当前 VDEC/VPSS/DRM 公共接口没有完整闭环，不作为近期移植目标 |

所以“解码器能输出隔行内容”“CPU 滤镜能 bob”和“独立 MCDEI 硬件已开放”是三件不同的事。

当前DEI是VPSS四场硬件处理，不等于原厂完整MCDI、运动估计、TNR或PQ链。旧`dei`/`dei_field_rate`模块参数已移除；DEI由显示重排后的`hivxedei`显式提交，不改变其他解码会话的capture布局。

当前`hivxe-1080p50`使用同一套尺寸/帧率策略：SD/HD保留原尺寸；逐行25/50保留帧率，25帧/秒隔行输入按场输出50p；超过1920×1080的逐行输入由VPSS硬件按比例缩小。码率/GOP不由分辨率名推断。编解码器语法中的picture cadence优先于开流时的packet/r_frame_rate猜测。

AVS+ MPEG-TS必须编入`cavsvideo` parser以合并跨PES的两场。VENC的压缩包使用独立CPU内存，立即回收有限的CAPTURE DMA缓冲，防止音频交错封装把编码队列全部占住；像素表面仍由DMA-BUF交接。这两类CPU复制不能混为一谈。

## 错误处理

硬件完成失败必须返回错误，不能用 repeat-last、插值或强制交付坏帧掩盖；
这份文档只记录公开接口和已确认的能力，不把寄存器字段、stub 或软件
fallback 当成硬件支持。

## 接口与转码选择

FFmpeg 未指定 `-hwaccel` 时默认走 CPU 软件解码，输出通常是 `yuv420p`。显式指定 `-hwaccel v4l2request` 才会尝试 HiVXE Request；硬件初始化或格式协商失败时，FFmpeg 可能回退到 CPU，应从日志确认最终选择的格式。

`/dev/mediaX` 中的 `X` 是命令模板占位符，不是固定编号，也不是
`/dev/dvb/adapterX` 的编号。media minor 按设备注册顺序动态分配，重启或
模块顺序变化后可能改变。当前 VDEC 的 media model 是
`HiSilicon Hi3798CV200 VDH`，可这样查找实际节点：

```sh
for node in /dev/media*; do [ -e "$node" ] || continue; b=${node##*/}; printf '%s: ' "$node"; cat "/sys/class/media/$b/name"; done
```

选择名称为 `HiSilicon Hi3798CV200 VDH` 的节点，并用
`media-ctl -d /dev/mediaN -p` 检查拓扑；系统存在 `/dev/media/by-path/` 时可
优先使用对应链接。将查到的实际 `/dev/mediaN` 或 by-path 路径写入 Spawn
profile；不要把字面量 `mediaX`、通配符或 `$MEDIA_NODE` 直接交给
Tvheadend，Spawn 字段不会替换 shell 变量。

## Tvheadend Spawn profile 模板

这些命令只保留已完成实机吞吐验证的 HiVXE 路径。组播加入后的前几秒可能从 GOP 中间开始，出现缺少参数集或参考帧；因此模板使用 `discardcorrupt` 丢弃启动坏包，不使用会立即终止进程的 `-xerror`。首个有效 VPS/SPS/PPS 和 IDR 后应持续稳定输出。

Tvheadend 不会自动生成本页中的 profile；需要转码时，在 Web UI 的 `Configuration -> Stream -> Stream Profiles` 中新增一个 `MPEG-TS Spawn` profile，再把一整行命令填入命令字段。

`hivxe-1080p50-dei` 和 `hivxe-4k50-1080p50` 只是便于识别的 profile 名称，不代表已有同名文件。

### 1080i50 → VPSS DEI → H.264 1080p50

需要匹配的FFmpeg和内核`VIDIOC_HISTB_VPSS_DEI`接口。旧模块参数已移除；`HISTB_V4L2REQUEST_DEFER_CAPTURE_WAIT=auto`只流水化逐行下采样，隔行capture仍先验证完成。设备fd、完成状态和DEI输出池随帧引用存活，覆盖decoder EOF。

```text
/usr/local/bin/hivxe-ffmpeg -hide_banner -loglevel warning -nostdin -init_hw_device v4l2request=hivxe:/dev/mediaX -hwaccel v4l2request -hwaccel_device hivxe -hwaccel_output_format drm_prime -threads 1 -extra_hw_frames 0 -f mpegts -probesize 2097152 -analyzeduration 5000000 -fflags +genpts+discardcorrupt -i pipe:0 -map 0:v:0 -map 0:a? -map 0:s? -vf hivxedei -c:v h264_v4l2m2m -num_capture_buffers 16 -b:v 6000000 -g 50 -c:a copy -c:s copy -fps_mode passthrough -muxdelay 0 -muxpreload 0 -flush_packets 1 -mpegts_flags +resend_headers -f mpegts pipe:1
```

### 4K50 Main10 → VPSS ZME → H.264 1080p50

可与上一节共享`-vf hivxedei`：逐行输入透传硬件帧，不额外去隔行或翻倍帧率。`MAX_CAPTURE_SIZE`是上限，不会把SD强制放大到1080。

```text
/usr/bin/env HISTB_V4L2REQUEST_DEFER_CAPTURE_WAIT=auto HISTB_V4L2REQUEST_MAIN10_NV12=1 HISTB_V4L2REQUEST_MAX_CAPTURE_SIZE=1920x1080 /usr/local/bin/hivxe-ffmpeg -hide_banner -loglevel warning -nostdin -init_hw_device v4l2request=hivxe:/dev/mediaX -hwaccel v4l2request -hwaccel_device hivxe -hwaccel_output_format drm_prime -threads 1 -extra_hw_frames 0 -f mpegts -probesize 2097152 -analyzeduration 5000000 -fflags +genpts+discardcorrupt -i pipe:0 -map 0:v:0 -map 0:a? -map 0:s? -vf hivxedei -c:v h264_v4l2m2m -num_capture_buffers 16 -b:v 12000000 -g 50 -c:a copy -c:s copy -fps_mode passthrough -muxdelay 0 -muxpreload 0 -flush_packets 1 -mpegts_flags +resend_headers -f mpegts pipe:1
```

### CPU 解码 + H.264 编码

没有硬件解码条件时，把 `-c:v cavs` 保留在输入侧即可；编码仍可使用 V4L2 M2M：

```text
/usr/local/bin/hivxe-ffmpeg -hide_banner -loglevel warning -nostdin -f mpegts -probesize 2097152 -analyzeduration 5000000 -fflags +genpts+discardcorrupt -c:v cavs -i pipe:0 -map 0:v:0 -map 0:a? -map 0:s? -vf format=nv12,hivxebob -c:v h264_v4l2m2m -b:v 6000000 -g 50 -c:a copy -c:s copy -fps_mode passthrough -muxdelay 0 -muxpreload 0 -flush_packets 1 -mpegts_flags +resend_headers -f mpegts pipe:1
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
| `hivxedei` | 按显示顺序显式提交VPSS四场DEI，隔行25帧/秒输出50p，逐行输入不翻倍，输出独立DRM PRIME/DMA-BUF表面 |
| `HISTB_V4L2REQUEST_CAPTURE_SIZE=1920x1080` | 请求 VDEC capture 使用 VPSS ZME 输出 1080p，而不是在用户态缩放 4K 帧 |
| `HISTB_V4L2REQUEST_MAX_CAPTURE_SIZE=1920x1080` | 保留更小源尺寸，超限逐行源由VPSS按比例硬件缩小；支持同一profile接收SD/HD/UHD |
| `HISTB_V4L2REQUEST_MAIN10_NV12=1` | 让 Main10 解码表面经 VPSS 转为 8-bit NV12，匹配 CV200 H.264 VENC 输入 |
| `HISTB_V4L2REQUEST_DEFER_CAPTURE_WAIT=auto` | 只对逐行下采样启用异步capture消费，保留原尺寸/隔行解码的错误验证；完成状态和fd有独立引用生命周期 |
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
- 组播输入可能从 GOP 中间开始；不要在 Tvheadend 长驻 Spawn profile 中加入 `-xerror`，否则起始的缺 PPS/参考帧会让进程退出。同步完成后的持续坏帧仍应视为故障。
- 对实际交错画面不要写 `bwdif=deint=interlaced`；progressive 标记会使它跳过画面，这里应保留 `deint=all`。
- `-fps_mode passthrough` 和输出元数据中的 `50/1` 只描述时间戳，不保证整条解码、反交错、编码链达到实时速度。

## 公开来源

本页的目标是快速说明“做了什么”和“从哪里查”，不重复展开实机记录、
本轮范围、成熟度表或每个未确认条目的长篇分析。

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
