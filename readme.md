## 深度学习经典论文笔记

本文用于记录阅读深度学习经典论文时个人的一些理解，便于后期回忆，欢迎交流。

---

## 基础架构

### 1.[AlexNet](https://blog.csdn.net/qq_38807688/article/details/84206655)

结构是 (Conv->Relu>Pool->Norm) * 2->(Conv->Relu) * 3->Pool->(Full->Relu->Drop) * 2 ->Full

some keys:

​	1.学习率在整个训练过程中手动调整的。我们遵循的启发式是，当验证误差率在当前学习率下不再降低时，就将学习率除以10。

​	2.卷积or池化时，步长s比核的尺寸size小,这样池化层的输出之间有重叠,提升了特征的丰富性。

​	3.提出了LRN层，[局部响应归一化](https://www.jianshu.com/p/c014f81242e7)，对局部神经元创建了竞争的机制,使得其中响应较大的值变得更大,并抑制反馈较小的。（可以理解为某个点除以它那个h,w处前后depth_radius通道处的平方和）

​	4.显存占用 = 模型显存占用 + batch_size × 每个样本的显存占用，所以节省显存的办法有降低 batch-size、下采样、减少全连接层（一般只留最后一层分类用的全连接层）

​	5.dropout的作用是增加网络的泛化能力，一般用在全连接层（因为参数多）

### 2[.VGGNet](https://zhuanlan.zhihu.com/p/42233779)

结构简洁，整个网络都用同样大小卷积核尺寸和最大池化尺寸，反复堆叠3x3小型卷积核和2x2最大池化层。且拓展性强，迁移其他图片数据泛化性好。

一般结构：nn.Conv2d -> nn.BatchNorm2d -> nn.ReLU

[**亮点1：train image和test image大小是不一样的，是通过在测试时更改最后的全连接层为卷积层实现的，也就是测试时网络是全卷积网络，此时计算方式是不变的，只是会因为test image大小大一些导致最后卷积出来的结果会是 a * a * class_num （而不是 1 * 1 * class_num） ，此时只要对每一个通道（即class_num 的a * a矩阵做一个平均就好，再最后对所有的通道做softmax），对应keys #3**](https://www.zhihu.com/question/53420266)

**亮点2：**最终模型的准确率在 **1. [256;512] train image** (在这个区间随机取一个数字并resize到这个大小进行训练)+ **2.dense eval** (即亮点1，用3个不同的Q取test image进行eval)+ **3.multi-crop**(对于每个Q，都进行随机裁剪并随机翻转) + **4.神经网络融合**（ConvNet Fusion）（融合了D和E，基于文中D和E网络的softmax输出的结果的平均进行预测，因为模型之间可以互补，最后的表现还可以再提升一点）时达到最高

some keys:

​	1.两次3 * 3卷积核相当于一次5 * 5卷积核，但3 * 3参数更少，且通过多级级联可以直接结合非线性层。非线性表现力更强。

​	2.[对于选择softmax分类器还是k个logistics分类器，取决于所有类别之间是否互斥。](https://blog.csdn.net/SZU_Hadooper/article/details/78736765)所有类别之间明显互斥用softmax；所有类别之间不互斥有交叉的情况下最好用k个logistics分类器。

​	3.测试图片的尺寸(Q)不一定要与训练图片的尺寸(S)相同，测试时先将网络转化为全卷积网络，第一个全连接层转为7×7的卷积层，后两个全连接层转化为1×1的卷积层。结果得到的是一个N×N×M的结果，称其为类别分数图，其中M等于类别个数，N的大小取决于输入图像尺寸Q，计算每个类别的最终得分时，将N×N上的值求平均，此时得到1×1×M的结果，此结果即为最终类别分数，这种方法文中称为**密集评估**。

​	4.宽度的增加在一开始对网络的性能提升是有效的。但是，随着宽度的增加，对网络整体的性能其实是开始趋于饱和，并且有下降趋势，因为过多的特征（一个卷积核对应发现一种特征）可能对带来噪声的影响。

### 3.[GoogleNet (Inception)](https://blog.csdn.net/u014061630/article/details/80308245)

**亮点1：有多个辅助分类器**，因为GoogleNet深度很深，为了解决梯度反向传播的问题，将中间的某几层的输出的loss乘以0.3加到最后的cost上，最后推断（inference）时，去除额外的分类器。

亮点2：一个Inception模块由多个子模块组成，如1×1卷积、3×3卷积、5×5卷积和maxpool。

some keys:

​	1.好的网络会把高度相关的节点连在一起，而某层的输出中，在同一个位置但在不同通道的卷积核输出结果相关性极高，一个1×1的卷积核可以很自然的把这些相关性很高，在同一个空间位置，但不同通道的特征结合起来。因此Inception中含有大量的1×1卷积。

​	2.一个Inception模块中其他3×3卷积和5×5卷积还可以继续用来提取特征。

​	3.更大的模型（更多的参数）会导致 计算时间过长、过度拟合训练集、对训练集数量要求过高。

​	4.训练了7个版本的GoogleNet ，它们具有相同的初始化权重，仅在采样方法和随机输入图像的顺序不同，最终的预测结果由这 7个版本共同预测得到，这样泛化能力更强，准确率更高。

​	5.**训练时**，一些模型在较小的crop上训练，一些模型则在大的crop上训练，这些patch的size平均分布在图像区域的8%到100%间，长宽比例在3/4和4/3间，同时有改变亮度、饱和度的操作（photometric distortions），resize时内插方法随机使用bilinear, area, nearest neighbor and cubic, with equal probability。

​	6.**测试时**，先将图片分别resize到256，288，320和352的长/宽，再取这样图片的左，中，右方块（在肖像图片中，我们采用顶部，中心和底部方块），对于取出的方块将采用4个角+中心224×224裁剪图像+resize到224×224，并最终复制一份镜像的版本，这导致每张图像会得到4×3×6×2 = 144的裁剪图像，最终softmax概率在多个裁剪图像上和所有单个分类器上进行平均得到结果。

### [4.ResNet](https://zhuanlan.zhihu.com/p/56961832) 

[作者视频讲解](https://zhuanlan.zhihu.com/p/54072011?utm_source=com.tencent.tim&amp;utm_medium=social&amp;utm_oi=41268663025664)

some keys:

​	1.注意连接求和的时候直接每个对应feature map的对应元素相加即可，注意是先相加再ReLU激活。

​	2.Res18、Res34用的残差块是两个卷积操作，而Res50、Res101、Res152用的残差块是先1×1卷积降维，再3×3卷积提取特征，再1×1卷积升维，我的理解是因为深度深了，需要1×1卷积控制通道大小，而先降维再升维是为了先浓缩特征（通道数少了可以减少卷积的计算量），待卷积提取新的特征后再放大为了和前面的x维度相同，方便相加。

​	3.一个细节：Res50、Res101、Res152每个stage之间的第一个残差块跳远连接时维度不同，需要做一个升维（论文图中的虚线）（对应代码中stride=2部分）。

​	4.**训练时**，和AlexNet、VGGNet一样先每张图片减均值；**数据增强：**利用VGGNet的多尺度处理，从区间[256, 480]随机取一个数 S，将原图resize到短边长度为 S，然后再从这张图随机裁剪出224 x 224大小的图片以及其水平翻转作为模型的输入，除此之外还用了AlexNet的PCA颜色增强（物体的特征不随光照强度和颜色的变化而变）；在卷积之后ReLU之前用了BN；网络初始化方法用[这篇论文](https://arxiv.org/abs/1502.01852)；所有的网络都是从头开始训练；优化使用SGD，batch size = 256，学习率初始值0.1，每当验证集误差开始上升LR就除以10，总迭代次数达到60万次，weight decay (L2正则化前面的参数)= 0.0001，momentum = 0.9；不使用dropout。

​	5.**预测时**，和AlexNet一样进行**TTA**，每张图片有10个裁剪；并且和VGGNet一样采用全卷积形式和多尺度TTA，最后对它们的模型输出值取均值即可。

总结：加了跳远连接后，每个残差块学到的东西会比没有跳远连接学到的东西少，或者可以说学到的东西更精细、精确，这样在深度很深时将学到的各个精细的成分组合起来会达到很好的效果。

### [5.DenseNet](https://zhuanlan.zhihu.com/p/82901676)

由3-4个dense block组成，每个dense block内部由多个Bottleneck layers组成，每个dense block之间是transition layer层，用于减小feature-map的数量和特征的h,w。

some keys:

​	1.与ResNet不同，在特征组合时不通过直接相加，而是用concate来综合特征，增加了输入的变异性并且提高了效率。

​	2.一个dense block中，后面的Bottleneck layers与前面所有的Bottleneck layers都有连接。

​	3.每个Bottleneck layers都是先1×1卷积，压缩通道到4k大小，便于3×3卷积运算，再3×3卷积输出通道大小为k。

​	4.Transition layers做的是BN+1×1Conv+2×2 avg pool下采样，减小feature-map的数量（由θ决定减少到原来的多少，可设为0.5），同时也减少了feature的宽度。

## 神经网络优化

### 1.Adam

含梯度下降、动量、Adagrad、Rmsprop的思想。

