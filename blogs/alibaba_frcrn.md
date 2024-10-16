> 本文介绍了ICASSP2022 DNS Challenge第二名阿里和新加坡南阳理工大学的技术方案，该方案针对卷积循环网络对频率特征的提取高度受限于卷积编解码器(Convolutional Encoder-Decoder, CED)中卷积层有限的感受野的问题，将阿里达摩院之前的[FSMN](https://arxiv.org/pdf/2102.08551.pdf)与发展自DCCRN/DCCRN的[CRN with CCBAM](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9414569)结合。本文提出了一种频率递归卷积循环网络(frequency recurrence Convolutional Recurrent Network, FRCRN)框架在卷积循环编码器结构的基础上利用前馈顺序记忆网络(feedforward sequential memory network, FSMN)以提高沿频率特征的表征能力。具体而言，在CRED的每个卷积层之后利用FSMN沿频率维度对三维特征图(feature map)进行频率递归以建模范围更广的频率相关性并加强语音输入的特征表示；在编码器和解码器之间也插入了两个堆叠的FSMN层以进行时序建模。FRCRN在复数域预测复值理想比掩模(cIRM)，并利用时频域和时域损失优化，在ICASSP2022 DNS Challenge中取得第二名。

论文题目：[FRCRN: Boosting feature representation using frequency recurrence for monaural speech enhancement](https://ieeexplore.ieee.org/document/9747578)
作者：Shengkui Zhao, Bin Ma (阿里巴巴), Karn N. Watcharasupat, Woon-Seng Gan (新加坡南洋理工大学)
![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430204414622-997315050.png)


## 背景动机
CRN结构尤其是DCCRN在语音增强领域取得了优异的性能，但是卷积核有限的感受野限制了对频率维度的长范围建模。本文受DCCRN+中频率相关性建模研究的启发，提出了FRCRN以提高沿频率轴的特征表示。FRCRN在每个卷积之后加入一个用于频率递归的且相比LSTM参数量更小的FSMN层对特征图的沿频率轴建模，卷积层和频率递归层构成卷积递归(convolutional recurrent, CR)块。通过在编码器和解码器中叠加多个CR块来形成CRED，从而不仅能捕捉局地的时间谱结构，还能捕捉长范围的频率相关性。不像之DCCRN+只专注于建模时序关系，本工作专注于改进编码器-解码器结构的整体特征表征。整个模块如DCCRN+一样采用复值网络并估计复值理想比值掩码(cIRM)，利用时频域和时域损失函数进行联合优化。

## 模型架构
模型处理的整体流程如下图，带噪信号经过STFT后送入网络，估计得到的cIRM与带噪复谱按复数规则相乘得到增强复谱，反变换得到增强语音。FRCRN主要由CRED和时序建模模块组成，其中CRED包括对称的编码器模块和解码器模块，两个模块都包含多个CR模块。时序建模模块由两个堆叠的复值FSMN (CFSMN)层组成，带CCBAM的跳跃连接(skip connection)连接编码器和解码器以促进信息流动。
![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430200746642-1305355535.png)

**CR模块**：由复值二维卷积层、复值BN、LeakyReLU和CFSMN层构成。复值二维卷积和复值BN操作可参看[DCUNet](https://arxiv.org/abs/1903.03107v1)或[DCCRN](https://arxiv.org/abs/2008.00264)论文。其中卷积层的kernel size在时间维和频率维上分别为(2,7)，stide为(1,2)，时间维通过补零保证因果性，频率维不补零，输出通道数均为128。
![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430200721377-1194745238.png)

CFSMN可以暂且当成LSTM理解，CR中的CFSMN就是将频率特征维当作torch中的seq_len维度，通道维度当成torch中的input_size维度。其操作如下(和原文略有不同是因为已将文中参数带入公式)：
Step1(置换操作，对每帧并行处理): $U_{r/i} \in \mathcal{R}^{C \times T \times F} -> U_{r/i} \in \mathcal{R}^{T \times F \times C}$ 
Step2(对当前帧): $S_{r/i} = U_{r/i}[t,:,:] \in \mathcal{R}^{F \times C} =\{s_{f_1}, \cdots, s_{F}\}$
Step3(FSMN层，共有两组，分别为$FSMN_r$和$FSMN_i$): 
$h_{f_i} = ReLU(W_{f_i} s_{f_i} + b_{f_i})$
$p_{f_i} = V_{f_i} h_{f_i} + v_{f_i}$
$s_{f_i} = s_{f_i} + p_{f_i} + \sum_{\tau=0}^{20}{a_{\tau} \cdot p_{f_i-\tau}}$
Step4(复值操作):$S = (FSMN_r(S_r)-FSMN_i(S_i)) + j(FSMN_r(S_i)+FSMN_i(S_r))$
**时序建模**：将编码器输出的实（虚）部特征图频率特征维度和通道特征维度拉直成一维，而后对时间维进行CFSMN
**CCBAM（个人补充）**：该模块参考了图像中的SENet并拓展到复制网络，即分别对通道维和语谱维做attention。通道维注意力机制是对特征做均值池化和最大值池化后经过两个线性层通过Sigmoid函数得到注意力得分；空间维注意力机制是对特征做完以上两种池化后通过Sigmoid得到注意力得分。示意图如下：
![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430203135202-1141104911.png)


**损失函数**：
![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430200624906-442925190.png)
![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430200640931-1941410291.png)
![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430200655591-93667017.png)
**模型参数**：编解码器中各有6个CR模块，时序建模中有两个CFSMN。帧长20ms帧移10ms，STFT点数为1920，按1-641，641-1282，1282-1921的频点索引将整个STFT谱分为三组并沿通道为拼接，即网络输入通道数为3。网络输出的cIRM为对于为1921。
## 数据与结果
共生成3000小时的数据用于训练和开发，其中30%带混响。信噪比在0~15dB之间随机选取
参数量10.27M，计算量12.30GMACS每秒
![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430201701313-1854759078.png)

![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430201609972-708785832.png)

在DNS2020和VB-Demand中表现优异，DNS排名第二

消融实验说明了CFSMN频率递归、CCBAM和时序建模的有效性
![](https://img2022.cnblogs.com/blog/1516697/202204/1516697-20220430201815491-895182924.png)
