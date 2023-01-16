# Abnormal_bahavior_detection基於時空信息的異常行為檢測
> https://www.paddlepaddle.org.cn/tutorials/projectdetail/4381424

異常信息檢測的難點

<img src ='https://ai-studio-static-online.cdn.bcebos.com/db11e3803a8a4464ab6ffc076d4f07e0015d1373e1d6489783271e98a15b584f'>



动作相关任务种类较多，下面就动作相关的任务进行描述总结，并给出适用场景和代表模型：

<img src= 'https://ai-studio-static-online.cdn.bcebos.com/f2368f95505c48b5bec847370fa907472253afa16a5a41d3a3b7130ee0e6e5bb'>

案例基於slowFast和Faster-RCNN模型，對視頻中的異常進行檢測

## SlowFast

用於視頻識別，SlowFast包含兩條路徑

- Slow pathway
- Fast pathway

Slow pathway運行低幀率，用於補助空間語義信息，Fast pathway運行高幀率。獲取精確的時間運動信息。通過降低通道數量，Fast pathway分支可以變成輕量的網絡，同時也能夠學習到視頻中有用的時域信息。



### 動機

SlowFast 受到灵长类视觉系统中视网膜神经节细胞的生物学研究的启发。研究发现，*这些细胞中约80%的都是P-cell，约15～20% 是 M-cell。M-cell 以较高的时间频率工作，能够对快速的时间变化作出响应，但是对空间细节和颜色不敏感。P-cell 则提供良好的空间细节和颜色信息，但时间分辨率较低，对刺激反应比较慢。*

SlowFast 与此相似：

- SlowFast 有两条路径，分别处理低帧率和高帧率；
- Fast pathway 用于捕捉快速变化的动作，但涉及到的细节信息较少，与M-cell类似；
- Fast pathway 是轻量的，与M-cell的占比类似

### 思路

<img src = 'https://ai-studio-static-online.cdn.bcebos.com/c5d6f3de72a54cc5a8468cd6dd740f72ff1e7790d5b2421ea91f6c262ac56d2c'>

如上图所示，一条路径用于捕获图像或稀疏帧提供的语义信息，以低帧率运行，刷新速度慢。另一条路径用于捕获快速变化的动作，刷新速度快、时间分辨率高，该路径是轻量级的，仅占整体计算量的20%。这是由于这条路径通道较少，处理空间信息的能力较差，但空间信息可以由第一个路径以简洁的方式来处理。

依据两条路径运行的帧率高低不同，作者将第一条路径称为“Slow pathway”；第二条路径称为“Fast pathway”；两条路径通过横向连接进行融合。



### Slow pathway

可以是任意在視頻幀上時空卷積模型，（如時空殘差網絡，C3D，I3D，Non-local模型）

關鍵在於對視頻幀進行採樣時，時間步長`T`較大，（只處理`T`幀中的一幀，作者建议 `T`的取值为 16，对于 30fps 的视频，差不多每秒采样 2 帧。如果 Slow pathway 采样的帧数是 T，那么原始视频片段的长度为 `T原xT`。）



### Fast pathway

#### 高幀率

Fast pathway 的目的為了在時間維度上有良好的特徵表示，Fast pathway的時間步長`T/a`較小，<u>**其中`a>1`是slow pathway和Fast pathway之間的幀率比**</u>



作者建议 `a`的取值为 8。由于两条路径在同一个视频上进行操作，因此 Fast pathway 采样到的帧数量为 `aT` ，比 Slow pathway 密集 `a` 倍。



#### 高時間分辨率特徵

Fast pathway 具有高輸入分辨率，同時整個網絡結構會運行高分辨率特徵。在最后的分类全局池化层之前作者没有采用时间下采样层，因此特征张量在时间维度上一直保持在 `αT`。



#### 低通道容量



Fast pathway 是一个与 Slow pathway 相似的卷积网络，但通道数只有 Slow pathway 的 *β* 倍，其中 β<1，作者建议 β 的取值为 `1/8` 。这使得 Fast pathway 比 Slow pathway 的计算更高效。



低通道容量可以理解为表示空间语义信息的能力较弱。由于 Fast pathway 的通道数更少，因此 Fast pathway 的空间建模能力应该弱于 Slow pathway。但 SlowFast 的实验结果表明这反而是有利的，它弱化了空间建模能力，却增强了时间建模能力。



#### 橫向連接

作者通过横向连接对两条路径的信息进行融合，使得 Slow pathway 知道 Fast pathway 在学习什么。作者在两条路径中的每个“阶段”上使用一个横向连接，由于两条路径的时间维度不同，因此在进行横向连接时需要通过变换对两条路径的维度进行匹配。最后，将两条路径的输出进行全局平均池化，并将池化后的特征拼接在一起作为全连接分类器层的输入



### 實例化

<img src = 'https://ai-studio-static-online.cdn.bcebos.com/ac7985614dba4474a54b1eaf9dc695dbbafc2a3f3f1f4133994fe88bb9ba9915'>

 **<u>`TxS^2`表示时空尺度，其中 T 是时间长度，S 是正方形裁剪区域的宽和高。</u>**



