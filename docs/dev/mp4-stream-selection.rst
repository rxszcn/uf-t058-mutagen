======================================
MP4 容器流选择修复交付说明 (t058)
======================================

:范围: ``mutagen/mp4/__init__.py`` 中 ``MP4Info`` 的轨道（trak）选择与
       属性提取逻辑；``tests/test_mp4.py`` 新增矩阵测试
:字段契约: ``bitrate`` / ``channels`` / ``sample_rate`` /
       ``bits_per_sample`` / ``codec`` / ``codec_description`` /
       ``length`` 的字段名与语义保持不变，下游统计口径无需调整

1. 问题根因
===========

``MP4Info.load`` 原先在 ``moov`` 下遍历 ``trak`` 时，直接用
``trak[b"mdia", b"hdlr"]`` 取处理器原子并读取 ``data[8:12]`` 判断是否为
``soun`` 音频轨，存在四类缺陷：

1. **流类型信息缺失时泄漏裸异常**：``trak`` 缺少 ``mdia`` 或 ``hdlr`` 时，
   ``Atom.__getitem__`` 抛出的 ``KeyError`` 不在该层捕获（仅在
   ``MP4.load`` 的兜底 ``reraise`` 里被包成 ``MP4StreamInfoError``），
   直接实例化 ``MP4Info`` 的调用方会收到与容器语义无关的裸 ``KeyError``。
2. **流类型信息长度不足时静默错位**：``hdlr`` 负载短于 12 字节时
   ``data[8:12]`` 返回空串或短字节串，与 ``b"soun"`` 比较恒为 ``False``，
   畸形轨被**静默跳过**，既不报错也不进入无音频兜底，结果取决于轨的排列
   顺序，行为不确定。
3. **选择结果对轨顺序敏感、契约不明**：代码依赖 ``for/break`` 的隐式行为，
   没有显式表达“混合流取音频、双音频取第一条、纯视频不报数”的契约，
   一旦后续轨块畸形（如视频轨的 ``stsd`` 被误标或布局错位），视频
   VisualSampleEntry 中的 ``width/height`` 字段会落在 AudioSampleEntry 的
   ``channelcount/samplerate`` 偏移上，解析出由画面尺寸拼出的“天文数字”。
4. **选中音频轨后不校验 ``mdhd``**：``soun`` 轨缺少 ``mdhd`` 时同样以裸
   ``KeyError`` 失败。

2. 修复后的明确契约
===================

选择逻辑收敛到 ``MP4Info._find_audio_trak``，返回第一条音频轨或 ``None``：

========================== ============================================
容器形态                    行为
========================== ============================================
纯音频（单/多音频轨）        取**第一条** ``soun`` 轨的属性
音视频混合（视频在前）       跳过 ``vide`` 轨，取音频轨属性
音视频混合（音频在前）       直接取音频轨属性；与上一种结果完全一致
双音频轨                    取物理顺序上的第一条 ``soun`` 轨
纯视频/无音频轨（含 mvhd）   音频属性保持零默认值，仅从 ``mvhd`` 解析
                            ``length``
无任何 trak / 无 mvhd        抛 ``MP4NoTrackError``（``MP4.load`` 仍按
                            既有设计吞掉该异常，返回默认 info 对象）
畸形轨块（缺 hdlr/mdhd、     抛 ``MP4StreamInfoError``，错误信息指明具体
hdlr 截断）                 原子；不再泄漏 ``KeyError`` 等裸异常
未知 handler 类型            与视频轨等同处理，跳过继续查找音频轨
========================== ============================================

顺序无关性由“先按 handler 类型过滤、再取第一个匹配项”的单一遍历保证，
视频轨与音频轨的相对位置不影响结果。

3. 无音频文件默认值的取舍与依据
===============================

对纯视频 / 无音频轨容器，``bitrate``、``channels``、``sample_rate``、
``bits_per_sample`` 保持类属性默认值 ``0``，``codec`` /
``codec_description`` 保持空字符串，依据如下：

1. **沿用既有类级默认值，不改字段语义。** ``MP4Info`` 本来就把这些属性
   初始化为 ``0`` / ``u""``，且文档与 ``AudioSampleEntry`` 均约定
   “``0`` means unknown”。无音频轨与“属性未知”在语义上等价，沿用零值
   不引入新的哨兵值，下游基于零值的判断（如 ``if info.channels``）不回退。
2. **零值不会污染统计口径。** 码率/带宽聚合若对零值求和或计数，零值是
   中性元素；若采用平均值，下游可用“该文件无音频轨”这一确定语义自行
   排除。任何非零猜测值都会被当成真实采样率/码率计入转码参数与带宽
   统计，正是本次事故的来源。
3. **``length`` 例外，仍从 ``mvhd`` 解析。** 容器时长是容器级属性，不依赖
   音频轨，纯视频文件仍可获得正确时长（上游 1.48.0 起即有此兜底，
   见 NEWS 中 ``MP4: Fall back to parsing length from mvhd``）。
4. **高层 ``MP4`` 接口不抛异常。** ``MP4.load`` 继续捕获
   ``MP4NoTrackError``，因此打开纯视频容器得到的是带零默认值的 info
   对象而非调用失败，与历史行为一致；直接使用底层 ``MP4Info`` 的调用方
   则可通过异常类型区分“无轨”与“轨块畸形”。

4. 畸形流块的错误语义
=====================

* 缺少 ``mdia.hdlr``：``MP4StreamInfoError("track has no media handler
  (mdia.hdlr): ...")``
* ``hdlr`` 负载不足 12 字节（无法容纳 4 字节 handler_type）：
  ``MP4StreamInfoError("truncated hdlr atom: ...")``
* ``soun`` 轨缺少 ``mdia.mdhd``：``MP4StreamInfoError("audio track has no
  media header (mdia.mdhd): ...")``
* 无 ``moov``、无 ``mvhd`` 等既有错误类型（``MP4StreamInfoError`` /
  ``MP4NoTrackError``）保持不变。

``MP4StreamInfoError`` 与 ``MP4NoTrackError`` 均为既有异常类，异常层级
（``MP4NoTrackError`` 是 ``MP4StreamInfoError`` 子类）未变。

5. 测试矩阵
===========

``tests/test_mp4.py`` 新增 ``TMP4StreamSelection``，共 19 个用例，覆盖：

* 五种容器形态：纯视频、视频+音频、音频+视频、双音频（两种排列）、纯音频
* 顺序无关性断言（视频在前/在后结果逐字段相等）
* 无音频默认值（含仅 mvhd、空 moov、高层 MP4 接口三个入口）
* 畸形流块：缺 hdlr、hdlr 四种截断长度、soun 缺 mdhd、畸形轨位于有效
  音频轨之前/视频轨之后、裸 KeyError 不逃逸、未知 handler 跳过、
  soun 轨 stsd 条目截断、soun 轨缺 stsd 的默认值兜底

既有音频夹具断言（has-tags.m4a 等的 channels=2 / sample_rate=44100 /
bitrate / codec）全部原样保留，全套 4074 个测试通过，无回退。
