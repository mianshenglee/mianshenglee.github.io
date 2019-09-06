---
layout: post
title: 查阅了十几篇学习资源后，我总结了这份AI学习路径
category: IT
tags: ai machinelearning deeplearning
keywords: 
description: 
---


tags: ai,machine learning,deep learning

---


> 一句话概括：想进入AI领域，需要学习的的东西很多，如果能在纷繁复杂的知识中找到一条合理的学习路径，少走弯路，那该多好，本文将试图找到这条路。

# 1 引言

作为一名想进入AI领域的程序员，上网搜一下人工智能，大量的知识涌出来，有AI发展，有机器学习，有tensorflow，有python等等，但对于需要学什么，怎么学还是没有明确的答案。可以想象自己是一名大学老师，需要开一门AI的课程，那么课程如何设置才能合理，有效率地让学生学到知识。我查看了十多篇学习方法和学习资源的文章，浏览了几十篇相关内容后，做了一个资源整合，整理出一条相对完整的学习路径。希望通过此总结，一方面可以让大家对进入AI领域有一个清晰的学习目标，明白学习内容，也可以根据此路径制定自己的学习计划。另一方面也可以激励自己按计划学习AI知识。

通过本文，可以收获以下AI学习路径，同时会给出相应的参考学习资料：

- 学习一门新技能的方法论
- AI人文科普
- 基础知识
- 编程语言
- 机器学习
- 初级项目实战深化知识
- 深度学习
- 高级项目实战或论文

# 2 方法论

关于学习一门新技能或新知识，学习方法很重要，好的学习方法可以少走弯路。首先，学习前需要先明确两个问题：是什么？怎么学？这三个问题概括说就是：学习目标与学习计划。学习目标比较清楚，就是踏入AI领域这个门，可以从事AI相关工作。学习计划就是对学习内容及过程的设计与执行，也就是本文所写的内容。还有就是建立学习的信心，学习不容易，以机器学习为例。在学习过程中，你会面对大量复杂的公式，在实际项目中会面对数据的缺乏，以及艰辛的调参等。只要制定合适的学习方法，学习是可以的。

明确了学习目标和计划，在学习的执行层面，则需要侧重于实践，以兴趣为先，践学结合。这里则特别提一下，使用费曼技巧，以教带学，是学习的好方法。简单来说，费曼技巧就是通过向别人清楚地解说某件事，来确认自己的确弄懂了某件事。它分为四个步骤：

1. 选择目标：明确目标选择一个概念
2. 教学：学习这个概念和相关知识，想象如何给一个孩子讲清楚。如果是真的讲授，更好。
3. 纠错并深入学习：教学过程中是否有不清楚的地方，如果有，继续学习，加深理解。
4. 简化类比：用自己的语言，简单的，通过和现实世界的实例关联类比，把一个概念讲清楚

根据费曼方法学习新技能，掌握更快，记忆更深刻。学习IT领域技能，此方法非常合适。

# 3 人工智能科普

## 3.1 AI人文历史

首先了解这个领域，建立起全面的视野，培养起充足的兴趣。AI是如何发展起来的，为什么在最近几年才成为热门的研究领域，AI技术包括哪些技术方向，有哪些应用领域，未来会如何发展，前景如何，对社会的影响如何等等，对这些问题都了解后，可以理解AI的前世今生，可以加深自己对AI的印象，加强对AI的兴趣，甚至可以发挥自己对AI的想象，对自己后续的AI学习可以有自己的想法。关于AI发展和科普，下面的资料可以参考：

- 书籍，《智能时代》，吴军
- 书籍，《智能革命》，李彦宏
- 书籍，《人工智能》，腾讯研究院
- 书籍，《人工智能简史》，尼克
- 书籍，《人工智能时代》《人人都应该知道的人工智能》，杰瑞卡·普兰
- 书籍，《科学的极致：漫谈人工智能》，集智俱乐部
- 书籍，《科技之巅》《科技之巅2》，麻省理工科技评论
- 博文，[从机器学习谈起](https://www.cnblogs.com/subconscious/p/4107357.html)：`https://www.cnblogs.com/subconscious/p/4107357.html`

## 3.2 当前AI发展及布局状况

要学习人工智能，先看看当前国内互联网巨头各自对AI的布局情况，就大概知道AI当前的风口在哪里，会有哪些重要应用，有哪些关键技术。各大公司旗下都设有AI平台的官网，[各大AI 开放平台一览](https://blog.csdn.net/qq_15071263/article/details/82908201)，地址:`https://blog.csdn.net/qq_15071263/article/details/82908201`，对各大AI平台的链接，可以看看。除了了解当前AI在各互联网公司的布局外，还可以关注一下这些公司对AI岗位的招聘要求及当前的各大招聘网站对此岗位的要求情况，这样有两个好处，一是明确自己的学习方向，学习有侧重点，二是做到对自己学习的一定的心理预期，知道自己学到哪个程度才能有机会获得此岗位。如下，是Boss直聘中的一则自然语言处理相关的招聘：

![招聘要求](http://ww1.sinaimg.cn/large/72d660a7ly1g6nwb6g1rpj20ii0bzdgq.jpg)

可见，数学基础、数据处理、自然语言处理、机器学习、数据挖掘等技术是比较关键的，也是学习的重点。

关于AI当前各大公司布局情况，参考资料如下：
- 文章，[各大AI 开放平台一览](https://blog.csdn.net/qq_15071263/article/details/82908201):`https://blog.csdn.net/qq_15071263/article/details/82908201`
- 网站，[百度大脑](https://ai.baidu.com/):`https://ai.baidu.com/`
- 网站，[腾讯AI开放平台](https://ai.qq.com/):`https://ai.qq.com/`
- 网站，[阿里达摩院](https://damo.alibaba.com/):`https://damo.alibaba.com/`
- 文章，[自动驾驶、金融、零售......BAT的AI之战打到哪儿了](https://www.huxiu.com/article/230094.html):`https://www.huxiu.com/article/230094.html`
- 书籍，[《人工智能标准化白皮书2018》](http://www.cesi.ac.cn/201801/3545.html):`http://www.cesi.ac.cn/201801/3545.html`
- 书籍，[《人工智能发展白皮书-技术架构篇（2018年）》](http://www.caict.ac.cn/kxyj/qwfb/bps/201809/t20180906_184679.htm):`http://www.caict.ac.cn/kxyj/qwfb/bps/201809/t20180906_184679.htm`
- 书籍，[《人工智能发展白皮书产业应用篇（2018年）》](http://www.caict.ac.cn/kxyj/qwfb/bps/201812/t20181227_191672.htm): `http://www.caict.ac.cn/kxyj/qwfb/bps/201812/t20181227_191672.htm`
- 书籍，[《中国信通院相关白皮书》](http://www.caict.ac.cn/kxyj/qwfb/bps/): `http://www.caict.ac.cn/kxyj/qwfb/bps/`

## 3.3 AI架构及职位选择

### 3.3.1  AI架构视角
人工智能从业务视角可以分为感知能力、认知能力和服务能力三个层次，两大应用方向，如下：
![业务架构](http://ww1.sinaimg.cn/large/72d660a7ly1g6nwagyjvpj20hg07rq6i.jpg)

人工智能技术视角，可以分为基础设施层、技术层和应用层。如下：

![技术架构](http://ww1.sinaimg.cn/large/72d660a7ly1g6nway1uwbj20j00bvwis.jpg)

### 3.3.2 AI职位选择

通过上面两个图，基本了解AI涉及的领域及技术的总体架构，结合前面的当前互联网巨头的布局，可以看出，在未来，对于基础设施层和技术层，基本上由大公司来掌控和布局了，可发展和深入开发的空间相对较小，个人若想参与这些的研发，则需要从底层的技术和算法学起，要求很高。而在应用层，则会有更多的发展空间，利用`AI+行业`或`行业+AI`的模式，结合已有的AI基础设施和AI技术，可以做出更多的应用。这既是个人发展的机会，也是创业公司的机会。

文章[《腾讯云总监手把手教你，如何成为 AI 工程师》](https://cloud.tencent.com/developer/article/1004751):`https://cloud.tencent.com/developer/article/1004751`，对AI工程师做了分类，按垂直领域分：有语音识别，图像视觉，个性化推荐等业务领域的AI工程师。按从事研发内容分则有

- 1)AI 算法研究

这类人大都有博士学历，在学校中积累了较好的理论和数学基础积累，对最新的学术成果能较快理解和吸收。这里的理论是指比如语音处理，计算机视觉等专业知识。AI算法研究的人主要研究内容有 样本特征，模型设计和优化，模型训练。样本特征是指如何从给定的数据中构建样本，定义样本的特征，这在个性化推荐领域中就非常重要。模型设计和优化是设计新的网络模型，或基于已有的模型机型迭代优化，比如CNN网络模型中 AlexNet , GoogleNet v1/v2/v3, ResNet等新模型的不断出现，另外就是比如模型剪枝，在损失5%计算精度情况下，减少80%计算量，以实现移动终端的边缘计算等等。模型训练是指训练网络，如何防止过拟合以及快速收敛。

- 2)AI 工程实现

这类人主要提供将计算逻辑，硬件封装打包起来，方便模型的训练和预测。比如：
    - 精通Caffee/TensorFlow等训练框架源码，能熟练使用并做针对性优化；
    - 构建机器学习平台，降低使用门槛，通过页面操作提供样本和模型就能启动训练；
    - 通过FPGA实行硬件加速，实现更低延时和成本的模型预测；
    - 在新模型验证完成后，实现在线平滑的模型切换。

- 3)AI 应用

侧重验证好的模型在业务上的应用，常见语音识别，图像视觉，个性化推荐。当然这也包括更多结合业务场景的应用，比如终端网络传输带宽的预测，图片转码中参数的预测等等。

综上所述，在选择职位和方向时，除非有比较好的数学和算法基础，建议从AI应用层面来选择，会更容易入手，发展机会更大。

本章的参考资料：
- 文章，[如何系统学习知识图谱](https://blog.csdn.net/hadoopdevelop/article/details/79455758):`https://blog.csdn.net/hadoopdevelop/article/details/79455758`
- 文章，[腾讯云总监手把手教你，如何成为 AI 工程师](https://cloud.tencent.com/developer/article/1004751):`https://cloud.tencent.com/developer/article/1004751`


# 4 基础知识

要学习人工智能，免不了要学习算法，学习算法，则需要数学基础。而在具体计算过程中很多时候需要矩阵计算，因此线性代数知识也是需要。对于数据的分类，分析等，还需要有概率和统计。很多时候人工智能追求的就是最优化问题，举个粟子，BP神经网络使用的权重迭代变化，计算当前权重值离最优值的函数为损失函数，迭代过程中通过求导来确定调大还是调小，这个求导得到的函数就是梯度，而这个迭代的过程就是梯度下降，在这个过程中，微积分知识也少不了。在学习过程中，经常会遇到需要查看的论文了解原理，或者查阅一些英文资料，因此英文知识也是需要的。以上，总结来说，需要以下几大基础知识：

- 线性代数:标量、向量、矩阵/张量乘法、求逆，奇异值分解/特征值分解，行列式，范数等
- 概率与统计:贝叶斯、期望与方差、协方差、概率分布(0-1分布、二项分布、高斯分布)、独立性与贝叶斯、最大似然和最大后验估计等
- 高等数学:微积分、链式法则、矩阵求导、线性优化、非线性优化(凸优化/非凸优化)以及其衍生的如梯度下降、牛顿法等
- 英文:常备一个在线英文词典，能够不吃力的看一些英文的资料网页

以下是一些参考资料：

- 书籍，《线性代数应该这样学》，Sheldon Axler
- 书籍，《概率论与数理统计》，陈希孺
- 书籍，《数学分析新讲》三册，张筑生
- 书籍，《深入浅出统计学》， Dawn Griffiths
- 书籍，《统计学习方法》，李航
- 书籍，《矩阵分析与应用》，张贤达
- 文章，[《机器学习理论篇1：机器学习的数学基础》](https://zhuanlan.zhihu.com/p/25197792)：`https://zhuanlan.zhihu.com/p/25197792`

# 5 编程语言

当前人工智能开发使用的最多的当属`python`了，当然，`java`，`c++`，`matlab`和`R`也有不少。刚开始学习，直接选择`python`即可。对于编程语言的学习，一个字，练。直接上机操作，主要分几个模块的学习，python基础（语法，函数，数组，类等等），python常用的库，python的机器学习库。以下是一些`pyhton`的学习资料以供参考：

- 教程，[《廖雪峰Python教程》](https://www.liaoxuefeng.com/wiki/1016959663602400):`https://www.liaoxuefeng.com/wiki/1016959663602400`
- 教程，[《Python100例》](https://www.runoob.com/python/python-100-examples.html):`https://www.runoob.com/python/python-100-examples.html`
- 文章，[《从零开始写Python爬虫》](https://zhuanlan.zhihu.com/p/26673214):`https://zhuanlan.zhihu.com/p/26673214`
- 视频，[《零基础入门学习Python》](https://www.bilibili.com/video/av4050443):`https://www.bilibili.com/video/av4050443`


# 6 机器学习知识

## 6.1 机器学习算法

需要明确，当前人工智能技术中，机器学习占据了主导地位，但不仅仅包括机器学习，而深度学习是机器学习中的一个子项。目前可以说，学习AI主要的是学习机器学习，但是，人工智能并不等同于机器学习。具体到机器学习的流程，包括数据收集、清洗、预处理，建立模型，调整参数和模型评估。基础则是机器学习的基本算法，包括回归算法，决策树、随机森林和提升算法，SVM，聚类算法，EM算法，贝叶斯算法，隐马尔科夫模型，LDA主题模型等等。这些网上已经有不少机器学习的教程，学习非常方便，在搜索引擎一搜索，机器学习的文章也非常多，只要坚持下去，结合后面的实践，学习应该不成问题。以下是一些参考资料：

- 书籍，《机器学习实战》，Peter Harrington
- 书籍，《机器学习》，周志华
- 书籍，《机器学习导论》，Ethen Alpaydin
- 书籍，《机器学习基础：从入门到求职》胡欢武
- 书籍，《数据之美》，吴军
- 视频，[《machine learning》吴恩达](https://www.coursera.org/learn/machine-learning):`https://www.coursera.org/learn/machine-learning`
- 视频，[《李宏毅机器学习2017》李宏毅](http://t.cn/RpO3VJC):`http://t.cn/RpO3VJC`
- 文章，[《机器学习Machine-Learning》](https://github.com/JustFollowUs/Machine-Learning):`https://github.com/JustFollowUs/Machine-Learning`

## 6.2 机器学习框架

了解机器学习的算法，还需要有一定的工具来实现，好在现在已经有很多工具可以使用，如tensorflow，Keras，Theano，matlab等等，现在tensoflow是机器学习的热门框架，入门可以深入学习它。以下是一些参考资料

- 书籍，《TensorFlow实战》，黄文坚
- 书籍，《Tensorflow：实战Google深度学习框架》，郑泽宇
- 视频，[《Tensorflow教程》莫烦](http://t.cn/RTuDxFT)：`http://t.cn/RTuDxFT`

## 6.3 数据集选择

"巧妇难为无米之炊"，使用机器学习来进行项目实践时，如果没有数据，就更不用说模型训练了。因此，获取数据集来做测试数据也是一个比较重要的工具，好在现在网上有不少的数据集可以获取，参考资料如下：

- [手写数字库MNIST](http://yann.lecun.com/exdb/mnist/):`http://yann.lecun.com/exdb/mnist`
- [图像处理数据COCO](http://mscoco.org/):`http://mscoco.org`
- [机器学习经典开源数据集](https://www.jianshu.com/p/83ebd261862a):`https://www.jianshu.com/p/83ebd261862a`
- [机器学习数据集哪里找](https://www.jianshu.com/p/abce3d177e45):`https://www.jianshu.com/p/abce3d177e45`

# 7 初级项目实践

在实践中学习，用一些小的示例来实现功能，用机器学习来解决一个实际的问题(如图像领域，识别狗，识别花等等)，把机器学习方法当作一个黑盒子来处理，选择一个应用方向，是图像（计算机视觉），音频（语音识别），还是文本（自然语言处理），推荐选择图像领域，这里面的开源项目较多。也可以上github找一下相关的开源项目来参考。

# 8 深度学习知识

深度学习是机器学习中的一个子项，它源于人工神经网络的研究，含多个隐藏层的多层感知器就是一种深度学习结构。学习过程中，需要对深度学习的概念进行了解，熟悉BP神经网络，CNN卷积神经网络，RNN循环神经网络等原理及应用。以下是一些参考资料：

- 书籍，《Deep Learning for Computer Vision with Python》，Adrian Rosebrock
- 书籍，《Tensorflow：实战Google深度学习框架》，郑泽宇
- 书籍，《深度学习》，伊恩·古德费洛
- 书籍，《Python深度学习》，弗朗索瓦·肖莱
- 书籍，《深度学习与计算机视觉》，叶韵
- 视频，[《Deep Learning》吴恩达](https://www.bilibili.com/video/av49445369):`https://www.bilibili.com/video/av49445369`
- 视频，[《Stanford CS231N 2017》李飞飞](http://t.cn/RTueAct):`http://t.cn/RTueAct`
- 视频，[《一天搞懂深度学习心得》李宏毅](http://t.cn/RTukvY6):`http://t.cn/RTukvY6`
- 视频，[《李宏毅深度学习2017》](http://t.cn/RpO3VJK)：`http://t.cn/RpO3VJK`
- 视频，[《 Deep Learning With Tensorflow》](http://t.cn/RTuDcjC)：`http://t.cn/RTuDcjC`

# 9 高级项目实践或论文

具备了较强的知识储备，可以进入较难的实战。两个选择，工业界的可以选择看开源项目，以改代码为目的来读代码；学术界的可以看特定领域的论文，为解决问题而发论文。或者可以参加`kaggle`竞赛，来验证一下，解决问题。到了这个阶段，就看个人的修行了。不过到了此阶段，回头看一开始的学习计划，基本已经达到目的了。最后，对于论文查询，就不得不提arXiv了，arXiv是个收集物理学、数学、计算机科学与生物学的论文预印本的网站。将预稿上传到arxiv作为预收录，可以防止自己的idea在论文被收录前被别人剽窃。因此arXiv是个可以证明论文原创性（上传时间戳）的文档收录网站。现今的很多科学家习惯先将其论文上传至arXiv.org，再提交予专业的学术期刊。以下提供两个工具可以使用：

- [arXiv官网](https://arxiv.org)：`https://arxiv.org`
- [arxiv论文查询](http://www.arxiv-sanity.com)：`http://www.arxiv-sanity.com`
- [带代码的论文查询](https://paperswithcode.com): `https://paperswithcode.com`

# 总结

通过查询并阅读了十多篇对人工智能的学习方法和学习资源的文章后，本文试图对这些资源进行整合，整理出一条相对完整的学习路径，每一个阶段都给出了相应的参考资料，有了资料，更重要的是需要去学习和实践，希望对自己的学习有一个明确的计划，也希望对想进行AI领域的同学有帮助。

# 参考资料

- [AI 学习路线：从Python开始机器学习](http://t.cn/AiRnyMyj):
`http://t.cn/AiRnyMyj`
- [大龄程序员的人工智能学习之路](http://t.cn/AiTfbOds):
`http://t.cn/AiTfbOds`
- [机器学习从抬脚到趴倒在门槛](http://t.cn/AiRnUwsx):
`http://t.cn/AiRnUwsx`
- [这位成功转型机器学习的老炮，想把他多年的经验分享给你](http://t.cn/AiRnyUUS):
`http://t.cn/AiRnyUUS`
- [推荐学习顺序](http://t.cn/RYjAQNc):
`http://t.cn/RYjAQNc`
- [关于机器学习，这可能是目前最全面最无痛的入门路径和资源](http://t.cn/AiRnUJGm):
`http://t.cn/AiRnUJGm`
- [普通程序员如何正确学习人工智能方向的知识](http://t.cn/R8Ttwug):
`http://t.cn/R8Ttwug`
- [腾讯云总监手把手教你，如何成为 AI 工程师](http://t.cn/AiRnUlEL):
`http://t.cn/AiRnUlEL`
- [自上而下的学习路线: 软件工程师的机器学习](http://t.cn/Rfc2ih1):
`http://t.cn/Rfc2ih1`
- [机器学习数据集哪里找](http://t.cn/AiRnU1L3):
`http://t.cn/AiRnU1L3`
- [机器学习知识体系](http://t.cn/AiRnUeRB):
`http://t.cn/AiRnUeRB`
- [完备的 AI 学习路线，最详细的资源整理](https://zhuanlan.zhihu.com/p/64052743):`https://zhuanlan.zhihu.com/p/64052743`

