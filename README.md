# 雷达图像预测未来降水-CIKM AnalytiCup Top1清华团队思路分享(附代码）

![image.png-165.7kB][1]

---------
    《基于雷达图像的短期降水预报》是由ACM顶级数据挖掘会议___CIKM___举办的数据科学竞赛。CIKM 2017以__“智慧城市，智慧型国家”__为主题，通过人工智能同各学科领域的交叉研究，通过技术手段有效管理城市。

    本次 ___CIKM AnalytiCup 2017___ 由__深圳气象局__与__阿里巴巴__联合承办，旨在提升基于雷达回波外推数据的短期降水预报的准确性。比赛共吸引了来自全球__1395__个团队，来自__清华大学__的__Marmot团队__(姚易辰，李中杰)在比赛中脱颖而出，在复赛中以绝对优势排名第一。本文摘录了他们团队解题方案的核心思路予以展示。

> 比赛官网：[阿里天池大数据平台](https://tianchi.aliyun.com/competition/introduction.htm?spm=5176.100066.0.0.773ef42f8FXDoN&raceId=231596)

> 完整解题方案及代码:[https://github.com/yaoyichen/CIKM-Cup-2017](https://github.com/yaoyichen/CIKM-Cup-2017)

![image.png-143.8kB][2]


---------

## 赛题目标

- 赛题提供__10,000__组的雷达图像样本。每组样本包含60幅图像，为过去__90分钟__内(间隔6 min,共15帧)，分布在__4个高度__(0.5km, 1.5km, 2.5km, 3.5km)上的雷达反射率图像。

- 每张雷达图像大小为[101,101]，对应的空间覆盖范围为__101×101km__。每个网格点记录的是雷达__反射率__因子值Z。反射率因子，表征气象目标对雷达波后向__散射__能力的强弱，散射强度一定程度上反映了气象目标内部降水粒子的尺度和数密度，进而推测其与降水量之间的联系。
> ![image.png-632.1kB][3]

- 目标：利用各个雷达站点在不同高度上的雷达历史图像序列，预测图像__中心__位于[50,50]坐标位置的目标站点__未来1-2小时__之间的地面总降水量，损失函数为降水量预测值与真实值的__均方误差__。

> ![image.png-106.2kB][4]

---------

## 算法架构
> 本次比赛的特点在于__时空序列__的预测，即给出了目标站点周围一定空间范围的历史信息，需要预测在站点坐标上未来的降水走势，因而搭建时空之间的关联特性为解决问题的重中之重。同时有别于一般的计算机视觉问题，此次比赛提供的气象图像，其沿着时空方向的演化规律会满足一定的__守恒律__及__连续性__限制，发现物理问题的特殊性并寻找对应的表征量也是解决问题的关键。

> 解决方案的流程分为前处理，特征提取，模型训练__三__个部分。__前处理__步骤中，完成局部图像的拼接，并通过SIFT描述子寻找时间方向的对应关系，获得云团运动的轨迹。__特征描述__中，将问题的特征归纳为3部分，分别为时间空间方向的矢量描述，云团形状的统计描述，及由云团轨迹外推得到目标站点的雷达反射率的空间图像描述。__模型训练__主模型采用了卷积神经网络CNN，图像部分采用2层卷积池化，随后将向量拉平到一维，即在全连接层与其余非图像类特征合并，共同输入到2个隐藏层的神经网络中。

> ![image.png-41kB][5]

---------

## 图像拼接

> 赛题给出的局部雷达图像，样本与样本之间并不完全独立，图像样本之间存在一定的重叠，可以通过模板匹配的方式寻找样本之间的__坐标__关联特性。通过样本之间的局部图像拼接，能够将一系列小范围的局部雷达图像恢复到空间更大范围的雷达图像，进而获得关于云团更加整体的特性。通过局部图像的拼接，能够获得如下两方面效果：

> 1. 为目标站点的时空轨迹追踪提供更大的空间延伸量。目标站点附近更大的空间图像范围，能够对应更长的__时间外推量__。

>2. 获得云团整体的结构，方便从更为__宏观__的视角提取特征描述云团形态。

> ![image.png-165.9kB][6]

> ![pin2.gif-174.9kB][7]
 
---------

## 轨迹追踪
> 根据流体力学中的__泰勒冻结__假设(Taylor Frozen Hypothesis)，流场中存在显著的时空关联特性，即可以认为雷达反射图中云团在短时间内趋向于在空间以__当地平均对流速度平移__，短时间内并不会发生外形或者反射强度的剧烈改变。即监测点_x_ 处在未来_τ_ 时刻后的雷达信号_f_，能够通过平均对流速度_U_，从当前时刻_t_ 位于坐标的_x-Uτ_ 的信号中体现：

> ![image.png-1.5kB][8]

> 为了寻找每个空间坐标对应的对流速度_U_， 可以通过SIFT描述子在一定时间间隔内，在空间坐标上的匹配，寻找相同关键点在较短时间间隔_δt_ 内像素的平移量_δx_,即得到空间每个位置处的对流速度。

> ![image.png-1.9kB][9]

> 下图给出了__相邻__两帧图像上，SIFT描述子及相应的空间匹配关系。其中圆圈大小对应了关键点的特征尺度，圆圈中的刻度方向表征其主方向。两帧图像的匹配连线基本平行，即全场以一个近似相同的速度作对流运动。

> ![image.png-322.2kB][10]

---------

## 特征提取
> 特征包含__时间外插反射率图像，时间空间的矢量，云团形状的统计描述__三部分。

> __时间外插反射率图像__：由上述的图像拼接及轨迹追踪后，能够定位出全场的速度__矢量__见下图。以泰勒冻结假设和关键点匹配追踪到未来1.5个小时流场速度矢量后，能够外插未来每个坐标点的运动轨迹，即能够推测出未来位于目标站点上方的云团，在当前时刻雷达图像上的空间坐标。 图中__白色__圆圈坐标点的云团，会在1.5小时由图中对流矢量的作用下，运动到__红色__目标站点上方。因此截取空间轨迹上白点周围41×41大小，3个空间高度(1.5km,2.5km,3.5km)的局部图像作为卷积神经网络的图像输入。

> ![image.png-182kB][11]

>__时间和空间特征提取__: 在时间和空间方向（高度方向）提取图像像素的统计值（平均值、最大值、极值点个数、方差等等），作为时空特征的描述输入CNN的全连接层。

> __全局云团形状特征提取__: 某些特定的云层形态会对应典型降水事件。从拼接后的全局图像中提取云团形状的整体形态特征，包含雷达反射率的直方图和统计类信息、云团运动速度和方向、加速度、流线曲率、SIFT描述子的直方图、监测点位置、检测点反射率与最大值比值等。

> ![image.png-161.6kB][12]

[12]: http://static.zybuluo.com/Jessy923/k84ihymh9riz3b4t3ec4f9xs/image.png

---------

## 训练模型
- 卷积层中图像的输入为时间外推得到目标站点附近41×41的空间范围，采用较大的空间图像输入，希望能够包含轨迹预测的误差以及测评目标在1小时内的总降水量。图像部分采用2层卷积池化，随后将向量拉平到一维，即在全连接层与其余非图像类特征合并，共同输入到2个隐藏层的神经网络中。

- 模型通过dropout防止过拟合，keep_prob取值为0.65，梯度下降采用的Adam优化算法。1200个迭代步后即达到稳定。

---------

## 总结
> 虽然此前参加过多次大数据竞赛，但初次涉足图像类比赛能够获奖也是非常之意外。本次解题方案并未使用ImageNet上较为流行的InceptionNet或者ResNet，即用深度的图像卷积网络来做训练。而是针对气象问题的特殊性，针对时间空间关联这一重要线索，采用传统的关键点提取SIFT方法与卷积神经网络CNN结合的形式预测目标站点的降水量。

> ![Rank.png-20.6kB][13]




> 由于思路的特殊性，团队在未做调参的情况下已经能够大幅领先其他队伍。未来会对气象业务有更多探讨，用大数据力量推动气象预报的发展。感谢天池大数据平台组织比赛，感谢深圳气象局提供比赛数据，感谢CIKM2017组委会。

> 最后欢迎大家对于现有解题方案提出宝贵意见。队伍成员的邮箱是:

> 姚易辰：yaoyichen23@163.com

> 李中杰: lizhongjie1989@163.com

> 完整解题方案及代码:[https://github.com/yaoyichen/CIKM-Cup-2017](https://github.com/yaoyichen/CIKM-Cup-2017)

[1]: http://static.zybuluo.com/Jessy923/yjrme4ex0yk17szix7f474uo/image.png
[2]: http://static.zybuluo.com/Jessy923/3bx4m8agc2lkgjikjkeqice6/image.png
[3]: http://static.zybuluo.com/Jessy923/g5z39b2lv88avj6r577272z5/image.png
[4]: http://static.zybuluo.com/Jessy923/ad2ays0gtnd9kf5fxd5kqh97/image.png
[5]: http://static.zybuluo.com/Jessy923/7c2waipyaxp3sg0s38yp1pgh/image.png
[6]: http://static.zybuluo.com/Jessy923/3feon792j8zjwrkhu1dxnrep/image.png
[7]: http://static.zybuluo.com/Jessy923/zibvk7ft3werbpxtxmoce8ka/pin2.gif
[8]: http://static.zybuluo.com/Jessy923/3o2949c5zhgedqk2qtopyhqd/image.png
[9]: http://static.zybuluo.com/Jessy923/xc5f2t0ktkz1baz4zu8otpc4/image.png
[10]: http://static.zybuluo.com/Jessy923/mwxwbewrprskzgkpifpwz789/image.png
[11]: http://static.zybuluo.com/Jessy923/4rgo9ru9b2hwgm6yqk72f277/image.png
[13]: http://static.zybuluo.com/Jessy923/1dqg9w65jqec1zy1sauxf3oz/Rank.png
