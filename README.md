# HiSilicon Hi3798CV200 HiVXE 编解码能力

本文对照 Hi3798CV200 原厂 BSP 与本项目 Linux 6.12.y 的编解码实现。解码、编码分开列出；每种格式先看 Profile / 特性状态，再看下方的实现说明与限制。

快速跳转：[解码](#解码) · [编码](#编码) · [后处理](#后处理) · [Tvheadend 配置](#tvh-profile) · [来源](#公开来源)

## 组件与接口

| 组件 | 作用 |
| --- | --- |
| VDEC / VDH | 压缩视频重建，即硬件解码 |
| VENC | H.264 硬件编码；CV200 生产配置没有 H.265 / AVS 编码器 |
| VPSS / MCDEI | 缩放、去隔行、降噪等视频后处理，不是解码器 |
| JPGD / JPGE | 独立 JPEG 解码 / 编码模块，不与 VDEC 共用接口 |
| V4L2 Request | 用户态按帧提交码流、语法控件和参考帧的 Linux 无状态解码接口 |
| V4L2 M2M | Linux 内存到内存接口，也用于 stateful 解码、编码和转换 |
| BSP | 海思原厂板级软件及私有驱动；原厂已实现不等于开源 Linux 已可使用 |

### Profile 与配置的区别

- **编解码 Profile**：如 H.264 Baseline / Main / High、HEVC Main 10，规定码流的语法范围；解码和编码分别判断。
- **编译与设备配置**：决定驱动、固件、FFmpeg 组件是否交付。源码有实现，不代表安装包已编入。
- **TVH Stream Profile**：决定播放请求调用哪个转码入口。名称中写 `720p` 或 `High` 不会自动修改分辨率或编码 Profile。

### Linux 接入路径

| 路径 | 接口与用途 |
| --- | --- |
| 无状态硬解 | V4L2 Request 使用 VDEC / VDH，按帧提交码流与控件；格式以解码矩阵为准 |
| 有状态硬解 | Stateful V4L2 M2M 使用 `V4L2_PIX_FMT_MPEG4`、`VC1_ANNEX_G/L`；内核管理解析、显示和参考状态，不是 Request 控件 |
| H.264 硬编 | `h264_v4l2m2m` 使用 VENC，输出 H.264 4:2:0 |
| JPEG 硬编 | JPGE 独立 V4L2 M2M 接口，不与 VDEC Request 节点混用 |
| JPEG 硬解 | `histb-jpgd` 独立 M2M 节点已枚举 JPEG 输入及 NV12 / NV16 / YUYV 输出；表解析、采样配置与用户态协商仍有缺口 |
| 视频后处理 | `histb-vpss` 的 VDEC 内部路径已有 tile-to-linear、ZME 和四场 DEI；独立 M2M、MCDI / TNR 及目标尺寸分支的限制见后处理章节 |
| CPU 软件路径 | 如 FFmpeg `cavs`，可单独选择；不计为硬件支持，也不是固定 `hivxe-transcode` 硬件滤镜链的自动回退 |

`/dev/videoN` 与 `/dev/mediaN` 的编号由探测 / 加载顺序决定，不是固定 ABI。应按驱动名称、设备能力和 media 拓扑识别节点，不能把一次运行的编号写死到所有系统。独立 VPSS M2M 节点与 VDEC 私有 DEI ioctl 也不是同一接口。

### 状态如何阅读

状态表保留三列：**Profile / 特性、BSP、6.12.y**；长说明放在下方两列表中，不合并成宽表。表头 `6.12.y` 均指本项目带 HiVXE 补丁的 Linux，不是原生上游内核。

| 状态 | 含义 |
| --- | --- |
| ✅ 已做 | 对应列已有实现：BSP 须有匹配 CV200 的产品配置和可达解析器 / HAL 或调用链；Linux 须已实现并开放对应硬件接口 |
| 🟨 未做 / 未完成 | 只有部分实现、条件配置，或尚未证明完整可用路径；Linux 也包括尚未移植 / 开放的项目，详见说明 |
| ❌ 不支持 | 当前 CV200 产品配置或接口合同排除、明确拒绝，或没有可用硬件实现依据 |

BSP 与 Linux 两列**独立判断**。BSP 已做不表示 Linux 有隐藏开关；部分工具可用不表示整个 Profile 已完成。“不支持”仅限当前 CV200 产品及本项目接口，不推断其他 SoC 或芯片物理上绝无可能。

✅ 表示实现状态，**不是全部样片、尺寸、工具组合或持续实时吞吐均已验收**。能力矩阵以 BSP 产品配置、解析器、HAL、实际调用链及 Linux / FFmpeg 公开接口为依据；CPU 软件路径、头文件枚举和安装包组件清单不能替代硬件支持证据。

## 解码

BSP依据以R005 SPC060的Hi3798CV200产品配置及其实际选中的语法/HAL为主。各项工具仍受所属profile约束；例如解码B图像不表示Baseline允许B slice，更不表示VENC能编码B帧。

Linux的H.264、HEVC、MPEG-2、VP8、VP9和AVS/AVS+使用V4L2 Request；MPEG-4 Part 2和VC-1使用stateful V4L2 M2M。CPU软件解码独立存在，表格不把它当成硬件能力。

### H.264 / AVC

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Baseline：8-bit 4:2:0 | ✅ | ✅ |
| Main：8-bit 4:2:0 | ✅ | ✅ |
| High：8-bit 4:2:0 | ✅ | ✅ |
| Constrained Baseline独立约束覆盖 | 🟨 | 🟨 |
| Extended（profile 88） | 🟨 | ❌ |
| High 10（profile 110） | 🟨 | ❌ |
| High 4:2:2 / High 4:4:4 | ❌ | ❌ |
| SVC Scalable Baseline / High（83/86） | 🟨 | ❌ |

**图像与工具**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
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

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| Baseline：8-bit 4:2:0 | CPU解码与V4L2 Request解码均有路径；硬件输出NV12。Baseline不包含后面明确拒绝的FMO等工具。 |
| Main：8-bit 4:2:0 | BSP和Linux有profile 77及对应语法/参考帧路径；与编码器Main支持分开判断。 |
| High：8-bit 4:2:0 | BSP和Linux有profile 100、8×8变换及scaling-list消息；不是High 10/4:2:2/4:4:4。 |
| Constrained Baseline独立约束覆盖 | 属于Baseline的约束子集，Linux按profile_idc=66处理；未据此声明独立约束识别及完整conformance。与VENC未开放Constrained Baseline菜单分开判断。 |
| Extended（profile 88） | BSP保留尝试解码的子集路径，遇data partition等工具仍受限；Linux公开校验不接收profile 88。不是VENC的Extended支持。 |
| High 10（profile 110） | BSP接纳profile ID、解析位深并报告非8-bit事件，不等于已证明10-bit重建/输出；不能把告警写成parser立即硬拒绝。Linux明确只接受8-bit。 |
| High 4:2:2 / High 4:4:4 | BSP SPS拒绝chroma_format_<wbr>idc>1，Linux也只接受4:2:0；profile名称或ID不足以扩大色度合同。 |
| SVC Scalable Baseline / High（83/86） | BSP有SPS扩展解析及部分失败分支，但未证明完整分层重建/输出；Linux没有公开SVC链路。 |
| I图像 | 按I slice提交；IDR及普通I图像的参考状态由DPB管理。 |
| P图像 | L0参考列表和运动预测进入VDH消息。 |
| B图像 | L0/L1参考列表与显示重排都有路径；不能据此声称VENC能编码B帧。 |
| CAVLC | PPS熵编码标志为0；包含已修复MB-pair终止地址的MBAFF路径。 |
| CABAC | PPS entropy flag、cabac_init_idc及CABAC硬件数据表已接入。 |
| P显式加权预测 | 亮度/色度权重、偏移和分母写入slice消息。 |
| B显式加权预测 | weighted_bipred_<wbr>idc=1，分别传递L0/L1权重。 |
| B隐式加权预测 | weighted_bipred_<wbr>idc=2交给VDH；实现不等于每种组合已实测。 |
| 多slice | 按编码顺序提交，要求first_mb递增；不承诺任意slice乱序。 |
| 长期参考与DPB | Linux验证VALID/<wbr>ACTIVE/<wbr>LONG_TERM/<wbr>FIELD；用户态管理MMCO和显示重排。 |
| MBAFF | 按MB-pair打包，输出完整woven NV12帧。 |
| 独立场图像 | 上下场配对、底场地址与参考语义已有实现；独立field样片覆盖不等于MBAFF验收。 |
| 8×8变换与scaling lists | PPS transform_8x8及量化矩阵都有实际消息写入。 |
| 去块滤波控制 | disable_deblocking_<wbr>filter_idc、alpha/beta偏移写入slice消息。 |
| 8-bit monochrome | BSP SPS接受chroma_format_<wbr>idc=0，但完整单平面重建/后处理输出尚未证明；Linux只接受chroma_format_<wbr>idc=1。 |

当前Linux不支持High 10、High 4:2:2、High 4:4:4、Extended、FMO、redundant picture、data partition、SP/SI slice和完整SVC分层解码。BSP的尝试解析路径不等于这些完整profile可用；CV200 SPS明确拒绝transform bypass。MVC另列，不从profile ID推断Linux双视角能力。

### HEVC / H.265

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Main：8-bit 4:2:0 | ✅ | ✅ |
| Main 10：10-bit 4:2:0 | ✅ | ✅ |
| Main Still Picture | 🟨 | 🟨 |

**图像与工具**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
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

**说明与限制**

| 项目 | 说明与限制 |
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

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| 8-bit 4:2:0 | ✅ | 🟨 |
| I图像 | ✅ | 🟨 |
| P图像 | ✅ | 🟨 |
| B图像 | ✅ | 🟨 |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| 8-bit 4:2:0 | BSP有MPEG-1分支；Linux现允许MPEG2_SLICE Request以零profile提交，FFmpeg也新增mpeg1_v4l2request。但MPEG-1专用硬件配置/消息语义未闭合，不能把接收请求或DC-only输出视作完整硬解。 |
| I图像 | 已复用MPEG-2 Request语法提交；硬件仍配置为MPEG-2，不据此升级完整MPEG-1 I图像支持。 |
| P图像 | BSP有前向参考与full-pel语义；当前FFmpeg Request控件未传递MPEG-1 full-pel标志，内核消息也未补齐对应语义。 |
| B图像 | BSP有双向参考语义；与P图像相同的full-pel/消息缺口仍在，接口存在不等于完整重建正确。 |

### MPEG-2

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Simple Profile：8-bit 4:2:0 | ✅ | ✅ |
| Main Profile：8-bit 4:2:0 | ✅ | ✅ |
| High Profile的4:2:0子集 | 🟨 | 🟨 |

**图像与工具**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
| I图像 | ✅ | ✅ |
| P图像 | ✅ | ✅ |
| B图像 | ✅ | ✅ |
| 逐行帧 | ✅ | ✅ |
| 独立场图像 | ✅ | ✅ |
| 量化矩阵与扫描顺序 | ✅ | ✅ |
| 显示重排与EOS/drain | ✅ | ✅ |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| Simple Profile：8-bit 4:2:0 | 只允许I/P；BSP/Linux均区分profile限制。 |
| Main Profile：8-bit 4:2:0 | 公开Request路径，sequence/<wbr>picture/<wbr>quantisation及参考帧均有控件。 |
| High Profile的4:2:0子集 | Linux validator接受profile 1且固定chroma_format=1；不据此宣称完整High或4:2:2支持，BSP的生产profile检查以Simple/Main为主。 |
| I图像 | 独立picture/slice及量化矩阵消息。 |
| P图像 | 前向参考消息，符合Simple/Main限制。 |
| B图像 | 用于Main，Simple Profile明确拒绝B图像。 |
| 逐行帧 | progressive_<wbr>sequence与picture flags一致性受校验。 |
| 独立场图像 | 上下场配对后输出完整NV12帧，不暴露单场像素格式。 |
| 量化矩阵与扫描顺序 | intra/non-intra矩阵、alternate scan及picture级工具进入消息。 |
| 显示重排与EOS/drain | 参考图像与捕获图像分别持有，排空剩余输出。 |

当前Linux只接受非scalable的4:2:0子集；4:2:2、spatial/SNR scalable不能由profile枚举或扩展头函数名推断为支持。

### MPEG-4 Part 2

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Simple Profile：8-bit 4:2:0 | 🟨 | 🟨 |
| Advanced Simple Profile：8-bit 4:2:0 | 🟨 | 🟨 |

**图像与工具**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
| I-VOP | ✅ | ✅ |
| P-VOP | ✅ | ✅ |
| B-VOP | ✅ | ✅ |
| QPEL | ✅ | ✅ |
| GMC / S-VOP | ✅ | ✅ |
| Uncoded VOP | 🟨 | ❌ |
| Resync / video packets | ✅ | ✅ |
| Packed bitstream | ✅ | ✅ |
| 隔行VOL/VOP | ✅ | ✅ |

**说明与限制**

| 项目 | 说明与限制 |
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

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Simple Profile | ✅ | ✅ |
| Main Profile | ✅ | ✅ |
| Advanced Profile | ✅ | 🟨 |

**图像与工具**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
| 逐行图像 | ✅ | ✅ |
| Advanced frame-interlace I/P/B | ✅ | ✅ |
| Advanced frame-interlace BI | 🟨 | ✅ |
| Advanced SKIP输出 | 🟨 | 🟨 |
| Advanced field-interlace | ✅ | ✅ |
| 逐行B / BI图像 | ✅ | ✅ |
| Bitplane / BPD | ✅ | ✅ |
| Intensity compensation | ✅ | ✅ |
| RESPIC | ✅ | ✅ |
| Main range reduction | ✅ | ✅ |
| Advanced range mapping | ✅ | ✅ |
| PSF | 🟨 | 🟨 |
| 场slice重复picture header | 🟨 | ❌ |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| Simple Profile | Linux Annex L解析profile 0，仅接受符合Simple限制的逐行工具组合。 |
| Main Profile | Linux Annex L解析profile 1且要求RES_RTM=1；不是V4L2 Request。 |
| Advanced Profile | Annex G主要帧/场路径已有实现，新增frame-interlace BI和SKIP分支；PSF后续语义及重复场slice头仍不完整，不能承诺完整profile。 |
| 逐行图像 | Simple/<wbr>Main/<wbr>Advanced分别遵循各自语法限制。 |
| Advanced frame-interlace I/P/B | 参考、bitplane和帧编码几何独立处理。 |
| Advanced frame-interlace BI | Linux已允许该变体进入BPD/消息/VDH路径；实现状态不代表所有组合已实测。 |
| Advanced SKIP输出 | Linux已接收SKIP并由CPU复制最近参考tile/PMV完成输出，不是新的硬件重建；不能将该混合路径计为纯硬件SKIP解码。 |
| Advanced field-interlace | 首场、次场、场事务及参考/输出生命周期有独立实现。 |
| 逐行B / BI图像 | 支持相应profile允许的图像类型；Simple不允许违反其B帧限制。 |
| Bitplane / BPD | bitplane解析、DMA、UP完成校验和参考语义都有路径。 |
| Intensity compensation | 参考映射、按parity组合和DMA表打包都有实现。 |
| RESPIC | 按当前coded macroblock网格计算BPD与重建区域。 |
| Main range reduction | Main语法及参考表面处理有路径。 |
| Advanced range mapping | 有捕获后range-map helper；Linux在CPU执行该后处理，不是纯硬件range mapping。 |
| PSF | Linux已读取并接受PSF标志，不再显式拒绝；但尚无后续专门语义处理依据，因此只标部分实现。 |
| 场slice重复picture header | Linux拒绝field slice picture_header_<wbr>flag=1；不要与已实现的独立场路径混淆。 |

### VP8

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Version 0 | ✅ | ✅ |
| Version 1 | ✅ | ✅ |
| Version 2 | ✅ | ✅ |
| Version 3 | ✅ | ✅ |

**像素格式与工具**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
| 8-bit 4:2:0 | ✅ | ✅ |
| Segmentation | ✅ | ✅ |
| 1/2/4/8 token partitions | ✅ | ✅ |
| LAST / GOLDEN / ALTREF | ✅ | ✅ |

**说明与限制**

| 项目 | 说明与限制 |
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

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Profile 0：8-bit 4:2:0 | ✅ | ✅ |
| Profile 1：8-bit非4:2:0 | ❌ | ❌ |
| Profile 2：10-bit 4:2:0 | 🟨 | ❌ |
| Profile 2：12-bit 4:2:0 | ❌ | ❌ |
| Profile 3：高位深非4:2:0 | ❌ | ❌ |

**图像与工具**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Frame context与概率更新 | ✅ | ✅ |
| Loop filter与segmentation | ✅ | ✅ |
| Tiles | ✅ | ✅ |
| Show-existing frame | ✅ | ✅ |

**说明与限制**

| 项目 | 说明与限制 |
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

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| AVS JiZhun Profile（0x20） | ✅ | ✅ |
| AVS+ Guangdian Profile（0x48） | ✅ | ✅ |

**像素格式与工具**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
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

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| AVS JiZhun Profile（0x20） | CPU cavs与V4L2 Request均有路径；不要与FFmpeg的无关avs解码器混淆。 |
| AVS+ Guangdian Profile（0x48） | 生产配置开启AVSPLUS，AVSP与VDH消息已有实现。 |
| 8-bit 4:2:0 | Linux要求sample_<wbr>precision=1、chroma_format=1，输出NV12。 |
| I图像 | sequence/<wbr>picture/<wbr>slice/<wbr>decode四组Request控件。 |
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

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Stereo High（profile 128）双视角 | ✅ | 🟨 |
| Multiview High（profile 118）双视角 | ✅ | 🟨 |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| Stereo High（profile 128）双视角 | BSP subset-SPS、inter-view reference及HAL消息可达；Linux已链接MVC helper并加入参考/消息挂钩，公开扩展NAL输入及双视角输出仍未闭合。 |
| Multiview High（profile 118）双视角 | CV200 BSP最多两个视角；不承诺任意N视角。 |

CV200 subset-SPS明确限制最多两个视角。当前Linux构建已链接 `histb-vdec-mvc`，有inter-view/DPB辅助逻辑，但未接通公开subset-SPS/MVC NAL摄入、view选择及双视角输出；FFmpeg现有H.264路径也未补齐dependent-view解码入口。接受profile 118/128或链接对象不构成完整MVC支持。

### RealVideo

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| RealVideo 8 / RV30 | ✅ | 🟨 |
| RealVideo 9 / RV40 | ✅ | 🟨 |
| RealVideo 10独立变体覆盖 | 🟨 | 🟨 |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| RealVideo 8 / RV30 | Linux已枚举RV30、链接helper，FFmpeg新增rv30_v4l2m2m；但RV被设为非Request格式后，核心stateful作业分发仍只接纳MPEG-4/VC-1，RV解析/HAL/输出链未接通。 |
| RealVideo 9 / RV40 | Linux已枚举RV40、链接helper，FFmpeg新增rv40_v4l2m2m；同样受核心分发缺口限制，不能把格式可枚举或备用hir9 stub算作正常解码。 |
| RealVideo 10独立变体覆盖 | 未发现独立RV10产品开关或明确的变体覆盖依据，不把所有RV40品牌变体自动算进来。 |

### VP6

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| VP6硬件解码 | ✅ | 🟨 |
| VP6F / VP6A独立变体 | 🟨 | 🟨 |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| VP6硬件解码 | CV200配置实际选择vp6语法与vdm_hal_vp6；Linux不开放VP6。 |
| VP6F / VP6A独立变体 | 不从泛型枚举或SCD处理函数推断完整flip/alpha解码与输出合同。 |

### JPEG / MJPEG（JPGD）

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
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

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| Baseline sequential，8-bit | BSP独立JPGD解析器/HAL有完整路径；Linux已新增独立节点，但仍使用默认量化/Huffman表，未将图像自身DQT/DHT完整写入硬件。 |
| Extended sequential，8-bit（SOF1） | BSP接受SOF0/SOF1并限制8-bit；Linux通用解析器可识别SOF1，但JPGD调用端未形成对应精度及硬件配置闭环。 |
| 4:0:0 / 灰度 | BSP识别单分量并配置YUV400；CV200有对应JPGD采样路径。 |
| 4:2:0 | BSP解析采样因子并向JPGD写入对应格式；可输出半平面YUV。 |
| 4:2:2水平采样 | BSP区分YUV422_21并配置采样/输出几何。 |
| 4:2:2垂直采样 | BSP区分YUV422_12，不与水平采样混为一个布局。 |
| 4:4:4 | BSP有采样因子、几何和JPGD格式路径；不借用VDEC/JPGE节点。 |
| Restart interval / DRI | CV200 BSP保留DRI支持；Linux成功解析时保存restart interval并写寄存器，但整体JPEG解码配置仍不完整。 |
| Progressive JPEG | BSP硬件资格检查拒绝progressive；Linux不提供此解码能力，且当前JPGD调用端吞掉解析错误，不能宣称已可靠拒绝所有不支持输入。 |
| Lossless / arithmetic JPEG | BSP拒绝对应SOF/DAC；Linux不提供此解码能力，通用parser的拒绝结果尚未由JPGD调用端完整传播。 |

Linux新增 `CONFIG_VIDEO_HISTB_JPGD` 和独立 `histb-jpgd` V4L2 M2M节点，公开单平面JPEG输入及NV12/NV16/YUYV输出。但输入采样仍未完整决定硬件配置，尺寸使用协商值而未与JPEG头闭合校验，NV16/YUYV仍沿用NV12的stride/sizeimage计算；因此不能将上述灰度、420、两种422、444及SOF变体整体升级为已完成。

FFmpeg虽有 `mjpeg_v4l2m2m` decoder，当前共享格式映射先选MJPEG，而该驱动仅枚举JPEG；现有精确匹配协商不会自动尝试另一个fourcc。该用户态路径尚不能视为与JPGD端到端接通，CPU MJPEG解码也不改变硬件状态。

### 未证实或未启用的硬件格式

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| DivX 3 | ❌ | ❌ |
| H.263独立硬解 | ❌ | ❌ |
| AVS2硬解 | ❌ | ❌ |
| AV1硬解 | ❌ | ❌ |
| PNG硬解 | ❌ | ❌ |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| DivX 3 | CV200配置条目不足以证明实现；实际仅hidivx3/hi_hal_divx3空函数/stub。 |
| H.263独立硬解 | CV200生产配置未启用，SOS明确关闭；软件适配器不是VDH路径。 |
| AVS2硬解 | CV200没有被产品配置选择的AVS2语法/HAL；不能套用其他SoC实现。 |
| AV1硬解 | 本次CV200产品及VDH路径没有实现依据。 |
| PNG硬解 | libpng/TDE软件与显示链不等于专用JPGD/VDH解码器。 |

AVS/AVS+硬件路径依赖17,920字节的`hisilicon/histb-avsp.bin`。缺少固件、长度不符或DMA约束不满足时，请求必须在启动硬件前失败。独立场、profile/尺寸变更和极端工具组合的实机覆盖不能从源码状态外推。

## 编码

CV200生产VENC只启用H.264与JPGE。H.264产品配置的尺寸范围为176..1920×144..1088、picture尺寸4对齐、帧率1..60；这些是产品边界，不是所有组合的实测吞吐承诺。standalone JPGE的BSP校验范围为16..4096，两条编码器不能共用能力结论。

以下profile行表示已实现的编码配置/语法路径。此前Baseline/Main/High分别有320×240、25 fps的最小硬件编码输出及解码检查记录；这些既有记录不覆盖本次新增格式、ROI、OSD、slice split及其全部组合。解码输入使用CABAC/CAVLC也不能代替编码profile验证。

### H.264 / AVC

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| Baseline：8-bit 4:2:0 | ✅ | ✅ |
| Main：8-bit 4:2:0 | ✅ | ✅ |
| High：8-bit 4:2:0 | ✅ | ✅ |

**图像与编码控制**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
| I / IDR图像 | ✅ | ✅ |
| P图像 | ✅ | ✅ |
| CAVLC | ✅ | ✅ |
| CABAC | ✅ | ✅ |
| 多slice编码 / slice split | ✅ | ✅ |
| 全局码率控制 | ✅ | ✅ |
| QP控制 | ✅ | ✅ |
| GOP控制 | ✅ | ✅ |
| 周期IDR / refresh-period控件 | 🟨 | ✅ |
| 帧率控制 | ✅ | ✅ |
| 编码尺寸设置 | ✅ | ✅ |
| VUI sample aspect ratio | ❌ | ✅ |

**输入格式与扩展**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
| NV12：8-bit 4:2:0 UV | ✅ | ✅ |
| 独立Y/UV地址输入 | ✅ | ✅ |
| NV21：8-bit 4:2:0 VU | ✅ | ✅ |
| Planar 4:2:0输入 | 🟨 | 🟨 |
| Semiplanar 4:2:2输入 | 🟨 | 🟨 |
| YUYV输入 | 🟨 | ✅ |
| UYVY输入 | 🟨 | ✅ |
| YVYU输入 | 🟨 | ✅ |
| ROI区域QP | 🟨 | 🟨 |
| OSD叠加 | 🟨 | 🟨 |

**未开放或不支持的编码配置**

| 特性 | BSP | 6.12.y |
| --- | --- | --- |
| B帧编码 | ❌ | ❌ |
| Extended Profile | ❌ | ❌ |
| Constrained Baseline独立配置 | 🟨 | ❌ |
| 独立区域码率控制 | ❌ | ❌ |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| Baseline：8-bit 4:2:0 | BSP接受profile 66；Linux实现对应SPS/PPS，使用CAVLC。 |
| Main：8-bit 4:2:0 | BSP接受profile 77；Linux有Main+CABAC配置路径，不暗示B帧编码。 |
| High：8-bit 4:2:0 | BSP接受profile 100；Linux默认使用High+CABAC，PPS关闭8×8 transform并使用默认矩阵，不代表启用全部High工具，也不承诺与BSP字节级相同。 |
| I / IDR图像 | 强制关键帧、GOP起点和SPS/PPS输出有实际路径。 |
| P图像 | 使用已重建的参考表面；当前VENC为I/P链路。 |
| CAVLC | Baseline的实际编码熵模式；不是解码器的CAVLC能力证明。 |
| CABAC | Main/High的PPS entropy flag和VEDU熵编码位均实现。 |
| 多slice编码 / slice split | Linux已开放mode及MAX_MB（1..65535）/<wbr>MAX_BYTES（512..65535）控件，并编程PICFG1和收集码流描述符链；不承诺任意运行中切换。FFmpeg `-slices` 已接入宏块模式，但当前计数计算未包含宏块宽度，不能承诺恰好N个slice；byte模式未由该选项暴露。 |
| 全局码率控制 | Linux已开放CBR/VBR模式及帧级反馈RC，VBR复用反馈并放宽至配置QP上下限；不宣称严格HRD-CBR或完整峰值/缓冲约束VBR。bitrate控件范围32768..52428800 bit/s；FFmpeg的 `-maxrate` 在此路径只影响模式选择，不传递该峰值上限。 |
| QP控制 | 全局I/P QP、min/max QP范围0..51；关闭frame RC可固定I/P QP。FFmpeg `-q:v 26` 设置等值I/P QP；该包装对非正值使用26，不能用 `-q:v 0` 选择固定QP 0。不是区域QP接口。 |
| GOP控制 | 控制I/P序列的关键帧间隔，不是帧率或B帧间隔。 |
| 周期IDR / refresh-period控件 | 标准period控件范围0..1000，0关闭；Linux当前按周期强制IDR并调整ME阈值，type仅保存。FFmpeg `-intra_refresh N` 已转发控件，但不等于已实现逐宏块循环帧内刷新，也不能承诺无IDR修复网络丢包。 |
| 帧率控制 | CV200产品范围1..60；Linux目前只接受整数帧率，30000/1001、60000/1001不在当前接口合同内；不代表所有尺寸/速率组合实测通过。 |
| 编码尺寸设置 | BSP有宽高属性和SPS crop语法；Linux raw队列宽16对齐、高2对齐，公开S_FMT及左上角crop selection，不提供任意位置或任意缩放。 |
| VUI sample aspect ratio | 所扫BSP自动SPS不写SAR；Linux的软件SPS生成及标准V4L2 SAR控件已实现。这是软件头处理差异，不是芯片物理能力差异。 |
| NV12：8-bit 4:2:0 UV | Linux公开NV12M多平面队列并支持DMA-BUF导入；pitch与可见宽度独立协商。 |
| 独立Y/UV地址输入 | BSP的SrcYAddr/<wbr>SrcCAddr与HAL分别写入Y/C物理地址；Linux允许不同DMA-BUF fd及非零source data_offset。导入的压缩CAPTURE通过exporter的begin/<wbr>end_cpu_access维护CPU访问，不能对no-map地址直接做cache操作。 |
| NV21：8-bit 4:2:0 VU | Linux公开两平面NV21M并配置VU selector；FFmpeg已有NV21软件帧到NV21M的映射/上传。不要将其当作单平面NV21队列或NV21 DRM PRIME零拷贝合同。 |
| Planar 4:2:0输入 | BSP有Cr地址及专门stride分支，但尚未实机验证该布局；Linux未开放。 |
| Semiplanar 4:2:2输入 | BSP通过C stride×2抽取色度行，输出仍是4:2:0；不是High 4:2:2编码。 |
| YUYV输入 | Linux已公开单平面packed 422输入，配置对应store模式/字节序；FFmpeg有YUYV422映射。输出仍为H.264 4:2:0。 |
| UYVY输入 | Linux已公开单平面packed 422及对应selector，FFmpeg有UYVY422映射；不扩展H.264输出色度。 |
| YVYU输入 | Linux已公开单平面packed 422及对应selector，但当前FFmpeg共享V4L2格式表缺YVYU映射，不能仅靠通用像素格式枚举认定已接通。 |
| ROI区域QP | Linux已有enable/delta-QP控件（−30..30）及固定中央矩形的寄存器写入，FFmpeg `-roi N` 非零时启用。内部虽有8区域结构/写入循环，尚无用户提交任意坐标、尺寸、绝对QP或8区域payload的接口；不能称为完整8区域配置。 |
| OSD叠加 | Linux已有enable/alpha控件和固定位置/尺寸写入，FFmpeg `-osd N` 仅转发开关/alpha；尚无叠加像素、DMA地址、stride及缓冲生命周期的公共提交路径，不能作为可用水印功能推荐。 |
| B帧编码 | CV200生产slice模型只有I/P，缺B slice、L1参考和编码重排合同。 |
| Extended Profile | BSP CheckPrivateAttr明确拒绝；Linux profile菜单也排除Extended。 |
| Constrained Baseline独立配置 | BSP有profile 66路径，但未证明独立约束配置；Linux profile菜单明确屏蔽Constrained Baseline，不能用FFmpeg的通用枚举代替驱动支持。 |
| 独立区域码率控制 | ROI中的区域QP不是每区域bitrate；没有相应公共RC接口。 |

### JPEG / MJPEG（JPGE）

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
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
| Planar YUV420输入 | ✅ | ✅ |
| Quality / 量化表 | ✅ | ✅ |
| 旋转 | 🟨 | 🟨 |
| Slice split | ❌ | ❌ |
| Progressive JPEG编码 | ❌ | ❌ |
| Lossless JPEG编码 | ❌ | ❌ |

**说明与限制**

| 项目 | 说明与限制 |
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
| Planar YUV420输入 | Linux公开单缓冲连续Y/U/V（V4L2_PIX_<wbr>FMT_YUV420）；Y stride按32字节对齐，U/V stride为一半。不是planar 422/444或三个独立DMA-BUF接口；FFmpeg上传仍要求源布局与协商stride一致。 |
| Quality / 量化表 | BSP Qlevel=1..99；Linux JPEG quality=1..100，使用自身量化表规则。 |
| 旋转 | Linux已开放V4L2_CID_ROTATE的0/90/180/270度及FFmpeg `-rotate`；但422输入90/270度时硬件改为420，SOF仍写原422采样，存在头/硬件配置不一致。因此保留部分实现状态，不能承诺所有输入格式/角度组合。 |
| Slice split | standalone JPGE资格检查拒绝SlcSplitEn!=0；寄存器字段不构成公开支持。 |
| Progressive JPEG编码 | 没有相应SOF2和多scan编码路径，不能以软件JPEG能力代替。 |
| Lossless JPEG编码 | 公开JPGE为8-bit baseline DCT编码，不提供lossless接口。 |

JPGE内核格式与FFmpeg可选格式须分别核对：当前FFmpeg缺NV24/YVYU映射，NV16仅映射到NV16M而非此JPGE的NV16。codec列出的pixel formats不保证每条协商/上传路径可用；DRM PRIME编码上传仍限NV12，不能外推到新增软件帧格式。

### 其他编码格式

**支持状态**

| Profile / 特性 | BSP | 6.12.y |
| --- | --- | --- |
| HEVC / H.265编码 | ❌ | ❌ |
| AVS / AVS+编码 | ❌ | ❌ |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| HEVC / H.265编码 | CV200 venc_make.cfg只启用H.264和JPGE，不能套用MV200的HEVC VENC。 |
| AVS / AVS+编码 | CV200没有对应VENC生产配置及硬件消息合同；AVS+转码输出是H.264。 |

VENC格式转换表还出现planar 422/444、NV42、400/411/410等名称，但实际H.264源图像路径没有证明这些布局的完整转换，部分采样类型为VENC_YUV_NONE；不列为已知可用输入，也不承诺monochrome、High 10、High 4:2:2或High 4:4:4输出。

## 后处理

后处理沿用同一组“已做 / 未做 / 不支持”状态；软件滤镜单独标明，不计作硬件能力。

**支持状态**

| 能力 | 6.12.y |
| --- | --- |
| VDEC tile-to-linear | ✅ |
| 内部ZME缩放核心 | ✅ |
| 解码输出尺寸协商 | ✅ |
| VPSS field-rate DEI | ✅ |
| 内置PQ参数与条件DBM | ✅ |
| CPU bob（软件） | ✅ |
| 独立VPSS M2M及packed 4:2:2 | 🟨 |
| 完整MCDI去隔行 | 🟨 |
| TNR/<wbr>SNR/<wbr>MCNR | 🟨 |
| 实时PQ与独立增强调节 | 🟨 |
| DEI与目标尺寸联动 | 🟨 |
| 旋转与多输出端口 | 🟨 |
| 完整HDR tone mapping / 3D链 | 🟨 |

**说明与限制**

| 项目 | 说明与限制 |
| --- | --- |
| VDEC tile-to-linear | `histb-vpss` 在VDEC内部完成tile到线性帧转换；新注册的独立M2M节点与此内部路径分开判断 |
| 内部ZME缩放核心 | 装载8-tap luma / 4-tap chroma多档系数并按比例选择；已有3840×2160 Main10到1920×1080 NV12输出记录，50 fps元数据本身不证明整链持续实时吞吐 |
| 解码输出尺寸协商 | capture只向下协商，宽4像素、高2行对齐，缩小比严格低于16:1；可同时完成Main10到8-bit NV12转换，不提供公开放大 |
| VPSS field-rate DEI | FFmpeg显示重排后，`hivxedei`按LAST/<wbr>CUR/<wbr>NEXT1/<wbr>NEXT2提交四场DEI并将独立DMA-BUF交给VENC；入口限本VDEC的NV12表面、最大1920×1088，不接受任意导入buffer或P010；必须匹配下述UAPI版本 |
| 内置PQ参数与条件DBM | 已装载含DEI/FMD、DB/DM/DR/DS、SNR/TNR区域的默认表及部分FHD覆盖；四场节点按宽>128、场高>64启用DBM。参数已存在不等于每个处理块均开启 |
| CPU bob（软件） | extra提供 `hivxebob`，不是VPSS硬件DEI；`bwdif`未编入当前交付FFmpeg，不能直接套用其命令 |
| 独立VPSS M2M及packed 4:2:2 | 已注册独立节点，OUTPUT/<wbr>CAPTURE枚举NV12、NV21、YUYV、UYVY；尺寸16对齐、范围16..1920×16..1088。但任务仍复用tile输入路径，NV12/NV21及YUYV/UYVY的字节序未分别配置，不能据枚举宣称线性格式双向转换已可用 |
| 完整MCDI去隔行 | 已增加运动缓冲轮转、BLKMV/RGMV/PRJ地址编程和 <code>HISTB_VPSS_<wbr>DEI_MCDI_EN</code>；仍沿用四场提交，不能据此称为已完成原厂五场MCDI。当前FFmpeg不设置该标志 |
| TNR/<wbr>SNR/<wbr>MCNR | 已增加RFR DMA缓冲及 <code>HISTB_VPSS_<wbr>DEI_TNR_EN</code> 条件下的TNR/SNR/RFR寄存器编程；当前FFmpeg未启用，历史读写和跨会话状态的完整降噪链未验收，不承诺MCNR |
| 实时PQ与独立增强调节 | VDEC已暴露brightness、contrast、saturation、hue、sharpness标准控件（0..100，默认50）；仅部分DEI参数写入消费brightness/<wbr>sharpness，且端口配置会重写亮度背景值，不能称为五项画质调节完整生效或完整DB/DM、dering、deshoot接口 |
| DEI与目标尺寸联动 | UAPI新增 `target_width` / `target_height`，0分别沿用输入宽/完整帧高；内核已将尺寸传入VHD0/ZME。目标边界、目标stride/缓冲容量校验及FFmpeg输出布局/选项尚未闭合，不作为任意尺寸缩放的可用接口推荐 |
| 旋转与多输出端口 | 当前DEI仍仅有一个destination并写VHD0；目标尺寸联动不是VHD1、并发多端口或一次产生多路分辨率。VPSS旋转也未开放 |
| 完整HDR tone mapping / 3D链 | 新增Main10→8-bit时的输出dither/round及附加控制位，但未接通HDR传递函数/元数据、BT.2020→BT.709矩阵和tone-mapping曲线；Main10不等于HDR10，不能宣称HDR转SDR已完成。MVC解析与完整3D显示链也分开判断 |

### 默认路径与版本边界

所以“解码器能输出隔行内容”“CPU 滤镜能 bob”和“独立 MCDEI 硬件已开放”是三件不同的事。

当前默认 `hivxedei` 仍只提交四场硬件DEI，不设置MCDI/TNR标志，也不请求目标尺寸；新增内核分支不等于默认转码链已启用这些功能。旧 `dei` / `dei_field_rate` 模块参数已移除；DEI由显示重排后的滤镜显式提交，不改变其他解码会话的capture布局。

**DEI UAPI兼容要求：** `struct histb_vpss_dei_request` 已由含 `reserved` 的32字节布局改为含两个目标尺寸字段的36字节布局；`_IOW` 编入结构体大小，因此 `VIDIOC_HISTB_VPSS_DEI` 编号也改变。旧头文件编译的FFmpeg不能直接配新模块，需同步安装新UAPI头并重新构建、部署匹配的用户态；仅替换 `.ko` 不构成去隔行链验收。

既有匹配版本的非安全Linux DEI节点使用CV200 `CTRL.prot=2`，并为IGBM/IFMD提供真实的32 KiB统计写回DMA区；DBM按原厂四场节点的尺寸条件启用。统计写回地址不能留为0。此前TVH Spawn记录输出1920×1080逐行50 fps（45秒抓取含2170张视频帧、43.4秒视频，输出解码无错误）；这不代替本次UAPI变更后的复验，也不证明UHD输入、ALSA时钟或完整MCDI通过独立验收。

AVS+ MPEG-TS必须编入`cavsvideo` parser以合并跨PES的两场。VENC的压缩包使用独立CPU内存，立即回收有限的CAPTURE DMA缓冲，防止音频交错封装把编码队列全部占住；像素表面仍由DMA-BUF交接。这两类CPU复制不能混为一谈。

### 数据完整性

硬件完成失败必须返回错误，不能用 repeat-last、插值或强制交付坏帧掩盖。本页只记录公开接口和已确认的能力，不把寄存器字段、stub 或软件 fallback 当成硬件支持。

<a id="tvh-profile"></a>

## Tvheadend 配置与参数说明

### 1. 安装入口

安装方法、系统要求及更新 / 备份注意事项见隔壁的 **[extra 仓库](https://github.com/HiSilicon-Development/extra#readme)**；本节从安装完成后的 TVH 配置开始，不重复安装步骤。

extra 提供 `hivxe-ffmpeg`、私有库及 `hivxe-transcode` 转码入口，**不会自动创建 TVH Stream Profile，也不安装内核**。设备须使用匹配板卡的 [HiVXE 内核 / 固件包](https://github.com/HiSilicon-Development/firmware/releases)，用户态与内核的 DEI UAPI 也须匹配。

### 2. 新建并使用 Stream Profile

1. 打开 **Configuration → Stream → Stream Profiles**。
2. 新增 **MPEG-TS Spawn**；部分界面显示为 `MPEG-TS Spawn/built-in`。
3. 按下表填写并保存。

| 字段 | 设置 |
| --- | --- |
| Name | 例如 `hivxe`，仅用于识别 |
| Enabled | 启用 |
| Command line | 填写下方脚本的绝对路径，不附加参数 |
| Mime type | `video/mp2t` |

**Command line：**

```text
/usr/local/bin/hivxe-transcode
```

保存后还需要：

- **允许用户使用**：确保对应 TVH 用户有权使用这个 Stream Profile。
- **播放时选择**：在客户端或播放请求中选择 `hivxe`；继续使用 `pass` 等其他 Profile 不会调用它。
- **允许访问硬件**：TVH 服务用户须能执行脚本、访问相应 media / video 节点；按系统 video 组或 udev 规则授权，组权限变更后重启服务。

TVH 将选中服务的 MPEG-TS 送入脚本的 **stdin**，再从 **stdout** 读取转码后的 MPEG-TS；**stderr** 用于日志。这里调用的是外部程序，不需要修改 TVH 内置转码器，也不需要在界面中粘贴 FFmpeg 长命令。

**Command line 不是 shell。** 不在这里填写环境变量、`$MEDIA_NODE`、管道、重定向或 `&&`；这些内容不会按 shell 语法展开。TVH 4.3 直接命令解析最多保留 63 个参数，调用脚本也避免长命令被截断。

### 3. 默认输出与修改位置

默认入口的处理顺序是：

**Request 硬解 → VPSS 输出 NV12 / 按需缩小 → 按需硬件去隔行 → H.264 硬编 → MPEG-TS 输出。**

| 项目 | 默认行为 |
| --- | --- |
| 视频编码 | H.264，当前驱动默认 High＋CABAC；只有 I / P 帧 |
| 视频码率 | 目标 8 Mbps，不是固定 TS 总码率 |
| GOP | 50 张输出图像；50p 时约 1 秒，25p 时约 2 秒 |
| SD / HD 逐行 | 保留源尺寸及 25p / 50p 帧率，不放大、不额外翻倍 |
| 1080i50 | 每秒 25 张隔行帧 / 50 场，经四场 DEI 输出 1080p50 |
| 超过 1080p 的逐行输入 | 按比例缩至 1920×1080 上限内；保留 50p 时间戳不等于保证持续实时 |
| HEVC Main10 | 按 10-bit 重建，再由 VPSS 转为 8-bit NV12；不是 HDR→SDR tone mapping |
| 音频 / 字幕 | 复制原 packet，不转 AAC，也不转换字幕格式 |

**TVH Stream Profile 的名称不控制输出参数。** 分辨率、码率、GOP 和 H.264 Profile 均由入口脚本传给 `hivxe-ffmpeg` 的配置决定；图像节奏以实际 picture cadence 为准，不以开流时的 packet / `r_frame_rate` 猜测代替。

默认入口的内容以 extra 仓库的 [`hivxe-transcode`](https://github.com/HiSilicon-Development/extra/blob/main/payload/bin/hivxe-transcode) 为准，本页只解释配置，不另放完整脚本或预设模板。

如需调整：复制入口脚本并保留可执行权限 → 修改副本中的环境变量 / FFmpeg 选项 → 将相应 TVH Profile 的 Command line 改为副本的绝对路径。更新 extra 会覆盖标准入口；自定义副本需自行同步上游变化。

**入口脚本没有 `$@` 参数转发。** 在 TVH 中给路径追加 `-b:v 6000000` 不会修改码率；外部设置的同名环境变量也可能被脚本内的 `export` 覆盖。修改应在脚本内完成，而不是在 TVH 字段末尾追加选项。

### 4. hivxe-ffmpeg 配置说明

以下解释脚本实际使用的设置。`HISTB_...` 是本项目 FFmpeg 扩展读取的**环境变量**，不是标准命令行选项或内核模块参数。FFmpeg **输入选项放在 `-i pipe:0` 前，编码 / 输出选项放在其后**。

#### 环境变量

**延后等待硬件完成**

```text
HISTB_V4L2REQUEST_DEFER_CAPTURE_WAIT
```

默认值为 `auto`：逐行且 capture 为 NV12 时，延后到消费者使用前等待硬件完成，使流水线可并行；包括原尺寸逐行 NV12，不只下采样。隔行或非 NV12 仍先等待；错误检查移到消费前，不是跳过错误。

只有精确值 `auto` 应用上述条件；其他存在的值会无条件请求延后等待。**设为 `0` 不是关闭**，关闭须删除相应 export 并 `unset` 变量。

**Main10 输出格式**

```text
HISTB_V4L2REQUEST_MAIN10_NV12
```

默认值为 `1`：优先选择 NV12 capture，经 VPSS 将 10-bit 转成 8-bit，供 DEI / H.264 VENC 使用。不启用 H.264 High 10 编码，也不提供 HDR tone mapping。

这个变量同样按**是否存在**判断，`0` 不是关闭。去掉 export 并 `unset` 后，Main10 要求 P010，不能继续沿用只接受 NV12 的 DEI / H.264 编码链。

**解码输出尺寸上限**

```text
HISTB_V4L2REQUEST_MAX_CAPTURE_SIZE
```

默认值为 `1920x1080`：超限时按可见宽高比向下缩小，小源保留 coded 尺寸，不放大。输出宽 4 对齐、高 2 对齐，驱动缩小比须严格低于 16:1；不减少 VDH 重建面的尺寸。需要最高 720p 时，在脚本中将该值改为 `1280x720`。

**固定解码输出尺寸**

```text
HISTB_V4L2REQUEST_CAPTURE_SIZE
```

默认未设置。例如 `1280x720` 表示请求固定 capture 尺寸，**优先于 MAX 上限**，不能超过 coded 尺寸，也不自动保持宽高比；只适合尺寸足够大的固定输入，不适合混合 SD / HD 频道的统一入口。

脚本中的前三个 export 会覆盖调用者同名设置，却**没有清除继承的固定尺寸变量**。若要统一按上限缩小，应在自定义脚本中清除它：

```sh
unset HISTB_V4L2REQUEST_CAPTURE_SIZE
```

#### 硬件解码与输入

| 设置 / Setting | 选项、作用与限制 |
| --- | --- |
| 设备绑定 | `-init_hw_device` 创建名为 `hivxe` 的 Request 设备，默认绑定 `/dev/media0`；完整设备值见下方，不自动寻找 VDEC |
| 硬解接口 | `-hwaccel` 设为 `v4l2request`，`-hwaccel_device` 设为 `hivxe`，复用上述设备；不是 stateful M2M decoder |
| 硬件帧 | `-hwaccel_output_format` 设为 `drm_prime`，保留硬件帧供 VPSS / VENC 通过 DMA-BUF 交接 |
| 线程与帧池 | `-threads 1`、`-extra_hw_frames 0` 请求一个解码线程，用户额外硬件帧增量为 0；参考池和调度仍可能另加帧 |
| 输入格式 | 输入侧 `-f mpegts`、`-i pipe:0`，从 stdin 读取 TVH 送来的 TS |
| 开流探测 | `-probesize 2097152` 为 2 MiB；`-analyzeduration 5000000` 为 5 秒预算，不是总启动延迟保证 |
| 时间戳与坏包 | `-fflags` 使用 `+genpts+discardcorrupt`，补生成 PTS、丢弃已标记损坏的 packet；不修复所有码流错误 |
| 视频选择 | `-map 0:v:0` 必须选择第一个视频，不复制其他视频或全部节目结构 |
| 音频 / 字幕 | `-map 0:a?`、`-map 0:s?` 选择所有音频 / 字幕；保留 `?`，允许相应流不存在 |

设备绑定选项的值为：

```text
v4l2request=hivxe:/dev/media0
```

默认脚本固定使用 `/dev/media0`；若注册顺序不同，应修改为实际 VDEC media 节点，**不是 DVB adapter 编号**。其 media model 为 `HiSilicon Hi3798CV200 VDH`。可选装 `v4l-utils` 后，用 `media-ctl -d /dev/media0 -p` 查看拓扑；有多个节点时逐个检查。`media-ctl` 是诊断工具，不由 extra 提供，也不必每次开流前执行。

#### 去隔行与 H.264 编码

| 设置 / Setting | 选项、作用与限制 |
| --- | --- |
| 去隔行 | `-vf hivxedei`：逐行透传、隔行按场输出；由滤镜显式提交，不再使用旧的 dei 模块参数 |
| 编码器 | `-c:v h264_v4l2m2m`：VENC H.264 硬编，未指定 Profile 时为当前驱动默认 High＋CABAC，仍只有 I / P 链 |
| 编码缓冲 | `-num_capture_buffers 16`：申请 16 个压缩码流 CAPTURE buffer，实际数量以驱动返回为准；不是解码 DPB 或原始图像池 |
| 目标码率 | `-b:v 8000000`：视频目标 8 Mbps，CBR-like 帧级反馈；不是严格 HRD-CBR，也不是固定 TS muxrate |
| 关键帧间隔 | `-g 50`：每 50 张输出图像的间隔，不是输出帧率；调整为 25 也不会改变尺寸或帧率 |
| 编码 Profile | `-profile:v` 使用数值 ID，见下表；与 TVH Stream Profile 名称无关 |

| H.264 Profile | 输出配置与限制 |
| --- | --- |
| Baseline | `-profile:v 66`，CAVLC；不是 Constrained Baseline 独立配置 |
| Main | `-profile:v 77`，CABAC；不启用 B 帧 |
| High | `-profile:v 100`，或保留默认；CABAC，当前不启用 8×8 transform，不代表所有 High 工具均开启 |

`hivxedei` 只接受本 Request 路径的 NV12 DRM PRIME 硬件帧。**固定脚本没有完整的软件自动回退链**；CPU 帧不能直接交给这个滤镜。源若把交错画面误标为逐行，自动 DEI 会透传，不能靠 GOP 或输出 `-r` 修复场标志。

<details>
<summary>更多编码选项与实现限制</summary>

新增选项须先核对已安装 FFmpeg 的 encoder help。源码有选项、驱动有控件、实际效果已验证，是三个不同条件；默认脚本未主动设置 ROI、OSD、refresh 或 slice split。

```sh
hivxe-ffmpeg -hide_banner \
  -h encoder=h264_v4l2m2m
```

| 设置 / Setting | 选项、作用与限制 |
| --- | --- |
| QP 范围 | 如 `-qmin 20`、`-qmax 40`；范围 0..51 且 min≤max，不是固定 QP 或区域 QP |
| 固定 QP | 如 `-q:v 26`，关闭帧级 RC 并设置相同 I / P QP，不保证目标码率；当前包装不能用 0 选择固定 QP 0 |
| VBR | 如 `-b:v 6000000` 配合 `-maxrate 8000000`；正 maxrate 与 bitrate 不同时选择 VBR，但 maxrate 数值本身未作为硬件峰值上限传递 |
| 分片 | `-slices N`（N>1）已接入，但 MAX_MB 换算遗漏宏块宽度，不能保证恰好 N 片；先单独验证码流，不作为已验收策略推荐 |
| 中央 ROI | 如 `-roi -6`；0 关闭，非零范围 −30..30。仅固定中央矩形的 QP 偏移，不接受坐标、任意 8 区域或独立区域码率 |
| 周期 IDR | 如 `-intra_refresh 50`；0 关闭，最大 1000。当前为周期强制 IDR / 阈值调整，不是循环宏块刷新 |
| 去掉字幕 | 删除 `-map 0:s?` 和 `-c:s copy`，不改变视频硬件链 |

不能用一个选项补齐完整 MVC / JPGD / RV / VP6 / MPEG-1 硬解链、完整 MCDI / TNR / SNR、任意 8 区域 ROI、H.264 B 帧或 High 10 / 4:2:2 / 4:4:4 编码、HDR tone mapping。

`-osd` 的 1..255 仅转发开关 / alpha，无图像像素和 DMA 提交路径，不作为可用水印功能推荐。`-bf`、`-profile:v 110` 等通用枚举也不能补齐驱动实现。JPEG 的 `-rotate`（0 / 90 / 180 / 270）属于 `mjpeg_v4l2m2m` encoder，不是 H.264 或 VPSS 旋转，且仍受前述 4:2:2 采样头不一致限制。

</details>

#### 音频、字幕与输出封装

| 设置 / Setting | 选项、作用与限制 |
| --- | --- |
| 音频 / 字幕复制 | `-c:a copy`、`-c:s copy`，不转码；字幕仍须适合 TS 输出 |
| 帧时间戳 | `-fps_mode passthrough`，不为固定帧率主动重复 / 丢弃滤镜输出，不强制 50 fps，也不保证实时 |
| 封装缓存 | `-muxdelay 0`、`-muxpreload 0`、`-flush_packets 1`，降低 mux 缓存并逐 packet 刷新；不能消除探测、解码重排、DEI 或播放器延迟 |
| TS 表 | `-mpegts_flags` 使用 `+resend_headers`，重发 PAT / PMT，不是 H.264 SPS / PPS 的同义词 |
| 输出位置 | 输出侧 `-f mpegts pipe:1`，向 stdout 写 MPEG-TS；诊断文字不能写入 stdout |
| 日志 | `-hide_banner` 隐藏 banner，`-loglevel warning` 保留 warning 及更严重日志；排障可改为 info / debug，并在另一终端运行 `hivxe-top` |
| 进程与交互 | `exec` 让 shell 被 FFmpeg 替换，便于 TVH 管理进程；`-nostdin` 只关闭键盘交互，不妨碍媒体数据从 stdin 输入 |

**音频默认保持复制。** 当前精简包虽有 AAC encoder，却未编入 `aresample`。例如 MP2 解码的 s16p 无法自动转成 AAC 所需的 fltp，直接改为 `-c:a aac` 并设码率已有失败记录，不能当成通用 AAC 转码开关；需要时先补齐带采样格式转换能力的匹配 FFmpeg。音频转换在 CPU 执行，不是海思硬件音频编码。

### 5. 适用范围与排查边界

#### 当前安装包的输入范围

下面区分的是**当前 extra 构建工作流中的组件选择**，不是重新判定芯片或内核能力。已安装二进制须核对版本；可用 `hivxe-ffmpeg` 的 `-hwaccels`、`-encoders`、`-filters` 及对应 encoder help 查看组件，列表存在不代替硬件运行验证。

| 输入视频 | 当前默认入口的范围 |
| --- | --- |
| H.264 / HEVC / MPEG-2 / AVS / AVS+ | 已编入对应 Request decoder / hwaccel，由输入码流选择；AVS+ 需 `cavsvideo` parser 合并跨 PES 的两场 |
| VP8 / VP9 | 内核已有 Request 路径，当前包未编入相应解码链 |
| MPEG-4 Part 2 / VC-1 | 内核已有 stateful M2M 路径，当前包未启用对应 decoder；固定 Request 入口不会自动切换接口 |
| MPEG-1 | 源码有 `mpeg1_v4l2request`，当前包未编入；部分 MPEG2_SLICE Request 路径的硬件消息语义也未闭合 |
| RV30 / RV40 | 源码有相应 M2M decoder，当前包未编入；内核虽枚举格式，stateful 分发仍未接通，换 decoder 选项无法补齐 |
| JPEG / MJPEG 解码 | 当前包只有 CPU mjpeg decoder，未编入 M2M decoder；独立 JPGD 及共享 fourcc 协商仍有缺口 |
| VP6 / 完整 MVC | 没有完整公开硬解链，不能靠添加选项启用 |

如需 CPU 软件解码，应另建软件 decoder / 软件滤镜处理链。当前包提供 `hivxebob`，没有 `bwdif`、`hwdownload` 或 `setfield`；处理误标为逐行的交错源也需先补齐相应组件，不能照抄未编入的滤镜命令。

#### 内核、固件与用户态须匹配

- 参考 defconfig 将 VDEC、VENC、JPGE 配为模块，对应配置为 `CONFIG_VIDEO_HISTB_VDEC`、`CONFIG_VIDEO_HISTB_VENC`、`CONFIG_VIDEO_HISTB_JPGE`；VDEC 选择 VPSS 和共用 SMMU。磁盘上存在 `.ko` 不代表节点、固件和 FFmpeg 均已就绪。
- 解码固件置于内核固件搜索路径的 `hisilicon/` 下，通常为 `/lib/firmware/hisilicon/`。H.264 / HEVC 使用 `histb-h264-cabac.bin`、`histb-hevc-cabac.bin`；AVS / AVS+ 另需 17,920 字节的 `histb-avsp.bin`。
- 不能用发行版普通 FFmpeg 替代本项目 Request / VPSS 扩展；DEI 还须满足前文的 36 字节 UAPI 兼容要求。更换内核后须重启，模块须匹配运行内核。

#### 资源、错误与吞吐

- Main10 转 NV12 不减少 VDH 的 10-bit 重建内存；capture 即使是 1080p，4K HEVC 仍按 4K 分配重建面。UHD HEVC 的解码 capture 队列上限为 8 个 buffer，与编码器申请的 16 个压缩码流 buffer 是不同队列。
- 解码完成状态、slice 范围及 SMMU 错误仍由内核校验。组播从 GOP 中间开流可能需要等待参数集 / IDR，但之后应持续输出；不能将持续坏帧当作正常启动，更不能用 repeat-last 掩盖故障。
- 1080i50→1080p50 已有 TVH Spawn 运行记录；这不替代 UAPI 变更后的复验。4K50 等组合还须区分帧率元数据、实际帧数与持续实时吞吐，未经核验的组合不作实时承诺。
- 日志写 stderr，不能混入 TS stdout；不要在正常长驻流中使用 `-xerror`。GOP、编码 buffer 数、H.264 Profile 和 TVH Profile 名称都不是帧率、解码池大小或能力启用证明。

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
  [JPGE](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/histb-jpge.c)、
  [JPGD](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/histb-jpgd.c)、
  [MVC helper](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/histb-vdec-mvc.c)、
  [RV helper](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/histb-vdec-rv.c)。
- **BSP语法与HAL交叉核对**：
  [H.264语法](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/32bit/core/syntax/h264.S)、
  [MVC语法](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/32bit/core/syntax/mvc.S)、
  [VP9 HAL](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/vfmw/vfmw_v5.0/product/Hi3798CV200/LNX_CFG/32bit/core/driver/HiVDHV4R3C1/vdm_hal_vp9.S)、
  [VENC属性校验](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/venc/venc_v1.0/drv_venc.c)、
  [VENC HAL](https://github.com/JasonFreeLab/HiSTBLinuxV100R005C00SPC060/blob/master/source/msp/drv/venc/venc_v1.0/hal_venc.c)。
- **Linux构建与后处理**：
  [HiVXE Makefile](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/Makefile)、
  [VPSS](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/drivers/media/platform/hisilicon/histb-vpss.c)。
- **DEI UAPI与用户态入口**：
  [histb-vpss.h](https://github.com/HiSilicon-Development/linux-6.12.y/blob/main/include/uapi/linux/histb-vpss.h)、
  [Request/DEI提交](https://github.com/HiSilicon-Development/FFmpeg/blob/9.0.1/libavcodec/v4l2_request.c)、
  [hivxedei](https://github.com/HiSilicon-Development/FFmpeg/blob/9.0.1/libavfilter/vf_hivxedei.c)。
- **FFmpeg编码选项与格式协商**：
  [VENC包装](https://github.com/HiSilicon-Development/FFmpeg/blob/9.0.1/libavcodec/v4l2_m2m_enc.c)、
  [JPEG包装](https://github.com/HiSilicon-Development/FFmpeg/blob/9.0.1/libavcodec/v4l2_m2m_jpeg.c)、
  [V4L2格式映射](https://github.com/HiSilicon-Development/FFmpeg/blob/9.0.1/libavcodec/v4l2_fmt.c)。
