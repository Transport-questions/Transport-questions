# 引言

传输之问的第一篇投稿，是mcc算法，本篇是我对该算法的交流与思考，采用第一视角。

我真诚希望我的思考对作者是有帮助的，所以，

为了得出客观的结论，本文分为纯人工无AI，gpt 5.6 sol无人工，以及交叉点评。

**第一期主题：网络传输的信号、博弈与数学**

投稿issue：

> 标题：测量对拥塞的贡献程度 ———— 在AS20473到AS17816的HTTP3优化实践
> 内容概要：
> 在AS20473到AS17816上，我运行有一个web服务器，注意到，http2流使用bbr，http3流使用NewReno变种拥塞控制，结果在夜间，http3流的速度极慢，由于quic-go上游多年积压拥塞控制相关issue，我自行fork并参照论文实现了bbr v1，随后主要出于对带宽利用率和改算法公平性不足为改进目标设计了我称为mcc的拥塞控制算法。
> 该算法核心思想，每个流保持合理的份额，全局自然公平。实践中采用一定时期的当前平衡RTT-近期最小RTT来判断对网络拥塞的贡献程度，通过一些判断逻辑， 如果自身的bdp超过了应有的份额，算法就会降低bdp上限，从而避免拥塞并保持尽量高的带宽利用率，如果其他流占用过高的份额，该数据流仍将保持不低于原有的发送速率，甚至一段时间后加速，只有当其他数据流让出超过其应得份额的带宽时，它才有机会调整自己的发送速率至当前实际交付速率。
> 在我的场景的，实现了比bbr v1更好的带宽利用率，流内部公平性较好，与外部流的公平性，特别是达到近似公平的收敛速度还需要改进。

Github主页：[qiulaidongfeng (qiulaidongfeng)](https://link.zhihu.com/?target=https%3A//github.com/qiulaidongfeng)

项目：[https://github.com/qiulaidongfe](https://link.zhihu.com/?target=https%3A//github.com/qiulaidongfeng/quic-go)

# 科普一下

作者的动机非常明确，是为了改善自己的网络性能。但QUIC-GO似乎并没有实现BBR，甚至，早期连拥塞控制都没有。

我看到这个issue时，第一反应就是BBR变种，也接近FAST TCP的思想。

FAST TCP也是一个基于RTT延时的算法，它为了保持公平性，也想到了让每个流保持一定的份额，但作者是怎么做的呢？我很好奇。

BBRv1有严重的公平性问题，而且很多地方存在不足，因为BBRv1的想法很简单，就是探测网络，然后基于延迟带宽积控制传输，并没有考虑多流共存问题。

为了吸引更多感兴趣的人，每一篇交流文章，我都尽可能介绍基础知识，并且把这些文章丢到对应的仓库里面。

## 面向读者的基础知识介绍

为了方便各位理解，我需要介绍一下拥塞控制的分类。

## 拥塞控制算法分类

我个人是把拥塞控制算法分为两类：

* 白盒建模
* 黑盒建模

为了方便理解，这里想象你有两根圆形水管，你往里灌水，水到达对端后，又沿着另一条水管过来：

这就是两个方向，TX和RX，那么：

容量：每秒最多通过C升水

往返时间：水往返一圈的时间

**什么叫黑盒建模呢？**

就是把网络当成一个黑盒子，尝试改变发送速率，看看这个黑盒子的输出有没有变化，如果有，那么就接收到反馈，调整自己的发送速率。

典型代码就是reno这些算法，它的思想很简单：

* 如果传输过程中没有出现丢包，那么就增加窗口
* 如果出现了丢包，那么就乘法减少窗口

很明显，这是一种试探的思想，如果速率一直增加，必然会触碰到容量上限，那么就会出现丢包，当算法看见丢包发生时，就知道出现了拥塞，必须减少速率。

**什么叫白盒建模？**

白盒建模则认为，我们是可以探测出网络的这些参数的，如果把丢包当成拥塞，那么在一个随机丢包的链路上，reno这些算法的性能是非常差的。

如果我们可以探测出容量上限、这条链路的往返时间，然后就可以按照固定的速率发包了。

比如我们可以探测出容量是1000b/ms，一个数据包来一次又回来耗时是3ms，那么，猜猜这根水管里有多少水？

可能有点抽象，让我们换一个比方：

你们高中每年招1000人，每名学生要读三年才能毕业，请问你们学校有多少人？

答案是3000人，水管这个也是同理，这被称作Little定律。

### Little定律

在排队论里，它是最基础的定律：

L=λW

其中：

- L：系统中的在途（in-flight）任务数量
- λ：系统的处理速率（throughput）
- W：任务在系统中停留的时间（delay）

## BBR思想

基于以上观察，BBR提出：为什么我们要依靠丢包作为信号，而不是探测出网络的带宽上限和往返时间呢？然后这样就能最大限度利用网络资源了。

当然，上面只是很简单的介绍，读者明白思想即可，详细内容可以查看相关论文和Linux源码的实现，Linux在4.9就支持BBR了，不过Linux高版本也没有实现BBRv3这些，而是BBRv1，这仍然有不可忽视的问题。

比如，BBRv1没有考虑公平性。

接下来让我们看看作者基于BBR上搞的MCC算法：

# 我的视角（无AI）

## MCC算法背景

> 我有一个使用AS20473网络的VPS，我在上面运行着我自己的web服务，同时支持HTTP2和HTTP3。我常在夜间进行开发，我使用AS17816的网络，我注意到，HTTP2协议的速度明显快于HTTP3。
>
> 我对vps进行过调大缓存区等优化，我仔细排查了所有操作，最后注意到，HTTP2传输层使用tcp，我设置拥塞控制使用bbr，HTTP3使用quic-go实现，我查看源码发现，它拥塞控制使用的是NewReno变种（初始拥塞窗口32，乘性减小因子0.7）。
>
> 由于上游多年积压关于bbr的issue，我自行fork实现了bbr v1，代码在https://github.com/qiulaidongfeng/quic-go/blob/master/internal/congestion/bbrv1.go ，感谢http://arthurchiao.art/blog/bbr-paper-zh/ ，高质量的翻译很大程度帮助了我。
>
> 随后我注意到，我的网络环境，夜间丢包率在2-10%，bbr v1有时在这类链路表现还有进步空间，主要是更高的带宽利用率和更好的公平性。bbr后续版本重新引入了基于丢包率的反馈，我决定按照这个链路的特点设计新的拥塞控制算法。

意识到问题，并愿意为了解决问题而去做一件事，这是非常可贵的。

作者在参考BBR的基础上重新设计算法，这个思路非常合理，尤其是考虑公平性问题，BBR v1虽然不把丢包当主要信号，但众所周知，其他的算法都是看丢包的，改善公平性，不可避免要考虑丢包。

但坦白来说，我认为BBR v1在抗丢包方面，应该是很强了，因为bbr v2把丢包的影响纳入了算法，所以我有点好奇，作者的BBR v1实现到了什么程度呢？这需要再开一篇文章了，不是本文的讨论范畴。

> ## 设计目标
>
> - 能在10%丢包的广域网运行
> - 保持高带宽利用率
> - 快速收敛到公平份额
> - 与其它拥塞控制算法近似公平
>
> ## 网络环境介绍
>
> - 发送端：AS20473 Vultr单核VPS 内核 6.12.95+deb13-amd64
> - 接收端：AS17816 Windows11
> - 链路情况：瓶颈带宽约160+Mbps，测得最小RTT240.13ms，通常平均RTT约280+ms

这个设计目标很明显，是针对高丢包链路的，同时考虑公平性？这恐怕有点难啊。

> ## MCC设计过程
>
> ### 设计思想
>
> 如果网络中每个流都能保持在公平份额，全局自然公平。然而，感知到其它流是需要时间的，不可能任何时候都在上帝视角做出完美的决定。
>
> 因此，设计转为更加实现更加现实的，**如果每个流都能对拥塞作出尽可能少的贡献，便可以控制拥塞，实现近似公平**。
>
> **这里的拥塞是按照bbr v1论文中的定义**，`拥塞（congestion）就是 inflight 数据量持续向右侧(大于测)偏离 BDP 线的行为， 而拥塞控制（congestion control）就是各种在平均程度上控制这种偏离程度的方案或算法。`

关于这个思路，我持观望态度，甚至，单纯靠端到端，是无法感知到其他流的。在我看来很多算法不可避免要试探拥塞，比如AIMD。

每个流贡献少，便可以控制拥塞？这句话我实在是有点没看懂，为什么要这么说呢？贡献于控制拥塞之间没有必然关系吧？

> ### 设计选取指标
>
> 丢包由于链路特性，我们不能将其作为参考。
>
> RTT能较好的反映一些链路的特性，因为选择此作为参考。
>
> 具体来说，bbr v1认为，**一段时间的最小RTT**`就是这条路径（在一段时间内）的往返传输延迟`，按照mcc的设计思想，我们认为，那就**近似一条流对拥塞没有做出贡献的基线**。
>
> 而**当前的RTT减去一段时间的最小RTT的差值，就反映一条流对拥塞的贡献程度**。后文简称差值。
>
> 为避免网络波动的影响，mcc使用当前的平滑RTT。

考虑作者的链路有较高丢包率，而RTT本身来到了200多ms，其实较高的RTT，往往意味着很多扰动会被抵消，因为基础RTT太大了，所以想达到2*RTT，普通的扰动是无法做到的。

看起来，作者认为拥塞是在开始排队时产生的，然后使用经典公式：
$$
RTT - RTT_m
$$
这个公式算出来是什么呢？其实就是中间数据包的排队时间。

作者认为这反映了一条流对拥塞的贡献程度？

这里怎么感觉怪怪的，是表述原因吗？排队是所有流共同造成的，因此这个公式反映的是整体拥塞情况，而不单单是某一条流。

> ### 如何判断对拥塞的贡献程度
>
> 最初，我试图将这种差值控制在一个绝对值，比如200ms，后来我注意到着不太能实现设计目标，因为改为不得超过1倍或2倍的最小RTT，称为阈值。主要是为了应对延迟ACK和适度利用缓冲区以提高带宽利用率。

这已经偏离了BBR的设计哲学了，BBR是不排队，而作者则是想控制排队。

> ### mcc的四个阶段
>
> #### mccStart
>
> 本质是慢启动。
>
> 初始BDP上限为128MSS，以便更快的探测瓶颈带宽，BDP上限呈指数级别增长。
>
> 平滑RTT因子试过1/8和1/4，结果1/4能更快的适应网络变化，加快收敛到近似公平，因此选择此。
>
> 值得说明的是，mcc中，实现的是quic-go中的拥塞控制接口，CanSend表示是否还有拥塞窗口，mcc的只要RTT差值没有超过阈值或没有接近bdp上限就可以继续发送，这似乎可以超出bdp上限。但这没有关系，HasPacingBudget控制pacing，如果当前交付速度达到上限，就不能发送。这种CanSend逻辑是为了更高的应对延迟ACK。
>
> 当测得对拥塞的贡献程度大时，进入mccStop。

很明显，这是参考了BBR，怎么说呢？BBR的这个设计，启动时太粗暴了。

> #### mccStop
>
> 类似快速恢复。
>
> 这个阶段，将BDP上限保持在当前，不改变，直到进入mccProbeRTT后切换到mccGuard,或测得不在对拥塞的贡献程度大，直接进入mccGuard。
>
> 这样做的目的，是避免被类似bbr v1的激进算法压制。

还是这个问题？每个流如何衡量自己对拥塞的贡献程度呢？这是不可知的问题。

> #### mccGuard
>
> 类似拥塞避免。
>
> 这个阶段，BDP上限重设置为根据当前实际交付速度和最小RTT计算的值，然后每秒增长1.25倍，这是为了在与bbr等算法的竞争中确保不被压制，避免像AIMD流在某些情况被抢走太多带宽。
>
> #### mccProbeRTT
>
> 每当超过5秒没有探测到新的最小RTT时进入此阶段，维持200ms，期间将BDP上限减半，并不得增长。

嗯，怎么说呢？这个拥塞避免是针对BBR的，这是怎么保证公平性的呢？从这些描述来看，我并不看好与AIMD的共存问题，很显然，虽然作者没有说明，但应该是存在隐患的，甚至同流的公平性我也有点担忧。

> ### mcc公平性理论
>
> 一般情况，mcc大多数时候差值没有超过阈值，当检测到超过时，意味着两种情况，自身占用远远超出瓶颈带宽的份额，或其它流加入，这两种情况，在进入mccStop后，都会保持原有的发送节奏，**但实际的交付速度不会超过瓶颈带宽，也不会比之前更快**。
>
> - 如果是新流加入，则新流加入会降低mcc流的交付速度，mcc在这个交付速度变慢的过程中让出超过公平份额的占用，并在退出mccStop时修正自己的BDP上限。
> - 如果是自身原因。mcc也将收敛至新的瓶颈带宽。
> - 如果遇到过于霸道的流，始终占用比公平份额更多的缓冲区，至多5.2秒后，再次进入mccProbeRTT，然后新的更高的最小RTT，使得mcc认为自身对拥塞的贡献程度不大，所以继续加速，这样即使面对STARTUP阶段的bbr流，mcc也会与bbr流进入其它阶段时实现动态收敛。
>
> 这种基于实际交付速度去修正BDP上限的过程反复发生，就是mcc实现动态收敛至公平份额的机制。

阈值是magic number，

至于同流公平性，这到底要怎么做呢？我很难想象，这里作者并没有给出证明，

作者也没有解释公平份额是从哪里来的，文档还是太简短了，不过好在有源码。

> ### 后续展望
>
> 在实测中，阈值1倍最小RTT的，我称为低延迟模式，它的带宽分配和最大延迟表现都较为优秀，但带宽利用率任有提升空间。
>
> 而阈值为2倍最小RTT时，它的带宽利用率令人满意，而且带宽分配近似公平，但部分流的尾部最大延迟极高。此外在多条mcc流占用大多数带宽时，给其它协议流让出带出的速度不够快。
>
> 如何修正两种阈值的不足，尚需思考。
>
> mcc实现在：https://github.com/qiulaidongfeng/quic-go/blob/master/internal/congestion/mcc.go
>
> 仓库issue可用，欢迎反馈。

怎么说呢？由于文档信息不足，我对公平性实现还是感到困惑，还是看看源码吧。

# 看源码

关于拥塞避免的如下：

```c
前面省略一些
	if b.state == mccGuard {
		b.ackCount++
		if b.ackCount%4 == 0 {
			b.pacing_rate += b.maxDatagramSize
		}
	}
}

func (b *MCCSender) mayExitProbeRTT() {
	if b.state == mccProbeRTT && time.Duration(monotime.Now()-b.last_probeRTTStart) >= 200*time.Millisecond {
		b.state = mccGuard
		if b.oldState == mccStart {
			b.state = mccStart
		}
	}
}
```

这确实是偏工程启发式的，跟我想象中的公平性有点不一样。

# 整体评价

很优秀的工程实现，作者在认识到现实世界的问题后，愿意去改变世界的不完美，这本身就是一种可贵的精神。

该算法吸收了RTT排队延时的想法，虽然理论分析不足，偏启发式，但这仍然是十分优秀的算法。

## 建议

其实这样就挺好，如果后续要往公平性这方面做的话，可以进一步深入了，

简单解释一下，一般证明协议公平性，我个人常用的就是计算长期吞吐，AIMD的吞吐公式是：
$$
\boxed{
B(p)
\sim
\frac{1}{\text{RTT}}
\sqrt{\frac{3}{2bp}}
}
$$
为了证明协议的公平性，一种思路是证明算法在与AIMD共存时吞吐公式相当，这需要一些数学推导。

如果没有这个打算的话，我觉得现在就挺好，可以尝试再添加探测机制摆脱调参。

# 欢迎投稿

接下来给出AI的评审，参考价值不大，这里插入一波广告：

这是我们传输之问第一篇除了我之外的投稿，我是非常开心的，同时，欢迎大家投稿！

Github地址：[投稿示例：拥塞控制xxx · Issue #1 · Transport-questions/Transport-questions](https://github.com/Transport-questions/Transport-questions)

这个投稿示例在issue里面，第一个：

![image-20260829213622118](C:\Users\el\AppData\Roaming\Typora\typora-user-images\image-20260829213622118.png)

# GPT 5.6 sol视角

我的提示词是：

> 你好，你是一个客观、冷静的拥塞控制算法交流者，接下来你需要参加一位朋友的拥塞控制算法交流活动，你需要做以下内容：1.阅读算法简介和文档，再阅读源码实现 2.想法、从代码实现、理论实现和工程实现等多维度客观给出你的交流观点，对算法的分析可以分为理论分析和工程实现两部分，并且给出你的整体评价和有效建议 ，最后，输出一份文档，markdown格式。   以下是相关链接， 其中有算法实现的相关源码地址：issue是[测量对拥塞的贡献程度 ———— 在AS20473到AS17816的HTTP3优化实践 · Issue #4 · Transport-questions/Transport-questions](https://github.com/Transport-questions/Transport-questions/issues/4)    算法文档是：[Transport-questions/第一期投稿/测量对拥塞的贡献程度.md at main · Transport-questions/Transport-questions](https://github.com/Transport-questions/Transport-questions/blob/main/第一期投稿/测量对拥塞的贡献程度.md)

输出内容稍微放后面一点点，先放AI和我交叉评审：

# AI对我的看法

鉴于AI废话太多，AI对我的文字的看法摘取前几段吧：

交叉评审已完成。结论是：你的稿件对作者态度公正，核心理论质疑成立，但目前更接近“有价值的概念评议”，还不是完整的源码技术评审。

最重要的判断：

- `SRTT - MinRTT` 只能表示总体额外时延或排队压力，不能直接归因于本流。这是人工稿最准确、最有价值的观点。
- “人工独立分析 + AI 独立分析”能降低单一视角偏差，但不能保证结论客观；建议改为“提高可审查性”。
- “quic-go 早期没有拥塞控制”不成立：2017 年已有 CUBIC，2019 年切换到 Reno。
- “BBRv1 没考虑公平性”“BBR 不排队”“高 RTT 会抵消扰动”“该公式是一般 AIMD 吞吐公式”等表述需要修正。
- 源码评审遗漏了 pacing 未真正使用 `nextSendTime`、私有 RTT 估计绕过 ACK-delay 修正、ProbeRTT 并非简单减半、包号空间冲突、ECN 删除时间戳等关键问题。
- “优秀的工程实现”“十分优秀的算法”超过了现有证据。更合适的定位是：具有真实场景和可运行实现的实验性拥塞控制原型。
- 对作者而言，当前稿件的概念反馈价值高，工程可操作价值中等；补充源码调用路径、可复现实验和公平性定义后，参考价值会明显提高。

对当前人工稿的总体判断是：**它是一份有价值的第一轮概念评议，但还不是完整的算法或源码技术评审。**

其中最重要、也最成立的判断是：

> `SRTT - MinRTT` 反映的是端到端路径上的总体额外时延或排队压力，不能仅凭这个量把拥塞因果归到当前流。

这一点正好击中了 MCC 名称与理论叙述中的核心缺口。你对“公平份额从何而来”“为何能够收敛”“同算法公平与跨算法公平应分别讨论”的质疑也成立。

当前稿件的主要不足有四类：

1. 对 Reno、BBR、FAST、Little 定律的科普存在若干过度简化或事实性错误；
2. 许多正确直觉用了“不可知”“很明显”“magic number”“不看好”等结论式措辞，但没有把成立范围写清楚；
3. 源码部分只检查了 Guard 的增长和 ProbeRTT 的退出，遗漏了会实质改变算法行为的实现问题；
4. “十分优秀的算法”“很优秀的工程实现”等总评，强度超过了当前证据能够支持的范围。

### 维度判断

| 维度               | 当前状态                 | 说明                                                         |
| ------------------ | ------------------------ | ------------------------------------------------------------ |
| 对作者的态度       | 公正、友善               | 能区分设计动机与理论疑问，没有进行人身归因                   |
| 核心信号判断       | 较强                     | 正确识别了“总体排队”与“本流贡献”的区别                       |
| 相关工作定位       | 部分准确                 | MCC 与 BBR、Vegas/FAST 都有关，但相关算法的描述过于简化      |
| 理论分析           | 有关键问题意识，但不完整 | 指出了不可识别性与证明缺失，尚未重建 MCC 的状态方程和收敛动态 |
| 源码审查           | 明显不足                 | 只引用了一小段代码，未覆盖 pacing、RTT、ProbeRTT、包号空间和 ECN 调用路径 |
| 实验证据审查       | 基本缺失                 | 未检查基线是否可比、重复次数、原始数据、统计区间和公平性指标 |
| 整体评价           | 偏高                     | 对工程动机的肯定被直接延伸为对算法成熟度的肯定               |
| 对作者的可操作价值 | 当前为中等               | 概念反馈有价值，但尚缺可以直接定位、复现和修正的工程问题     |

（顺便说一句，AI破事真的多，早知道不开max模式了，废话多到喜欢挑刺的地步了）

# AI视角意见

> 评审对象：MCC（Measuring the Contribution to Congestion）  
> 评审日期：2026-08-29  
> 评审性质：基于公开文档、指定提交源码、quic-go 调用路径及相关拥塞控制研究的独立技术交流意见

# 我对AI意见的看法

鉴于AI废话太多，我就把AI的意见的看法提前放这了：

我觉得AI说得还行的地方是：

* 拥塞贡献是全部流的，并不是单独流的
* 公平性这一块确实不足，甚至怎么证明同流公平性呢？

我觉得AI说废话的地方：

* 工程实现这里，废话又多又长，到处都是在检点细枝末节，感觉像是为了应付交流要求随便挑的

综上，前面部分可以看看，工程实现那些可以不看，为了材料完整，我还是选择保留：

## 一、评审范围与结论摘要

本次评审阅读并核对了以下材料：

- [投稿 Issue #4](https://github.com/Transport-questions/Transport-questions/issues/4)
- [算法文章《测量对拥塞的贡献程度》](https://github.com/Transport-questions/Transport-questions/blob/main/%E7%AC%AC%E4%B8%80%E6%9C%9F%E6%8A%95%E7%A8%BF/%E6%B5%8B%E9%87%8F%E5%AF%B9%E6%8B%A5%E5%A1%9E%E7%9A%84%E8%B4%A1%E7%8C%AE%E7%A8%8B%E5%BA%A6.md)
- Issue 指向的 [quic-go 提交 `2947a072`](https://github.com/qiulaidongfeng/quic-go/commit/2947a072a3396baaf7843f49a7349a52774e470a)
- 该提交中的 [MCC 源码](https://github.com/qiulaidongfeng/quic-go/blob/2947a072a3396baaf7843f49a7349a52774e470a/internal/congestion/mcc.go)
- quic-go 的发送门控、ACK、丢包、ECN、pacing deadline 等调用路径

整体结论如下：

1. MCC 源于一个真实且明确的工程问题：在约 240 ms 基础 RTT、2%—10% 夜间丢包的跨自治域链路上，避免 NewReno 因随机丢包持续退让，同时提高 HTTP/3 的带宽利用率。这一动机成立，目标场景也足够具体。
2. MCC 当前实际使用的主要拥塞信号是“平滑 RTT 相对近期最小 RTT 的增量”。这个量可以作为**总排队压力的代理信号**，但不能单独识别“本流对拥塞的贡献”。因此，算法名称和理论叙述目前强于信号本身能够支持的含义。
3. 当前公平性说明属于机制直觉，不是已经成立的收敛证明。特别是 `mccStop` 会在已有交付率处冻结，`mccGuard` 的增长近似与自身 ACK 速率成比例；这两个机制都没有直接消除流之间已有的速率差。
4. 源码已经接入 quic-go 拥塞控制接口，可以编译；但当前 pacing、RTT 采样、ProbeRTT、初始窗口等关键路径与文档叙述存在实质差异。在修正这些差异以前，实测结果不能完全归因于文档描述的 MCC 控制律。
5. 现有文章适合作为“算法设计与实践经验交流稿”。若要把结论提升为“能够测量拥塞贡献、快速收敛到公平份额、与 BBR/CUBIC 近似公平”，还需要形式化定义、源码修正和可重复对照实验。

## 二、算法的准确重建

为了区分文档意图与代码行为，先将源码写成控制变量和公式。

记：

- $M$：`maxDatagramSize`；
- $R_0$：`backGroundRTT`；
- $\bar R$：`smoothedRTT`；
- $x$：源码中的 `pacing_rate`；
- $g$：`bdpLimitFactor`，普通模式为 3，低延迟模式为 2；
- $F$：当前 `bytesInFlight`；
- $\widehat D$：最近一个平滑 RTT 窗口内的 ACK 交付率。

### 2.1 初始化

源码初始化为：

$$
R_0=5\ \mathrm{s}
$$

$$
x=\frac{128M}{R_0}
$$

`GetCongestionWindow()` 返回：

$$
W=xR_0
$$

因此初始报告窗口确实约为 $128M$。但是，后续 `CanSend()` 并不始终使用 $W$ 作为硬上限。

### 2.2 交付率估计

源码保留最近一个 $\bar R$ 时间窗口内确认的字节数：

$$
\widehat D(t)=
\frac{
\sum_{k:\ t-t_k\leq \bar R} a_k
}{\bar R}
$$

其中 $a_k$ 是某个被确认 QUIC 包的字节数，$t_k$ 是其 ACK 事件时间。它是 ACK 到达窗口上的平均交付率，不是 BBR 的逐包 delivery-rate sample。

### 2.3 RTT 阈值

源码中的布尔量 `r` 为：

$$
r=\left(\bar R-R_0\leq(g-1)R_0\right)
$$

等价于：

$$
r=(\bar R\leq gR_0)
$$

因此：

- 低延迟模式在 $\bar R>2R_0$ 时认为差值过大；
- 普通模式在 $\bar R>3R_0$ 时认为差值过大。

对文档给出的 $R_0\approx240.13$ ms 链路：

- 低延迟模式允许约 240.13 ms 的排队增量，触发点约为 480.26 ms RTT；
- 普通模式允许约 480.26 ms 的排队增量，触发点约为 720.39 ms RTT。

这也解释了文档中普通模式尾部最大延迟较高的观察。

### 2.4 实际发送门控

`HasPacingBudget()` 的允许条件是：

$$
\widehat D<gx
$$

`OnPacketSent()` 同时计算名义包间隔：

$$
\Delta t=\frac{L}{gx}
$$

其中 $L$ 为包长。

`CanSend()` 的代码行为则是：

$$
F<128M \quad\Longrightarrow\quad \text{允许发送}
$$

ProbeRTT 状态中：

$$
F<0.5gW
$$

其他状态中，如果 $r$ 为真，`CanSend()` 直接允许发送；只有 $r$ 为假时才应用：

$$
F<(g-0.5)W
$$

所以，当前代码中的发送控制不是单一的“BDP 上限”，而是以下三者共同作用：

1. RTT 条件门控；
2. ACK 交付率门控；
3. 部分状态下的在途量门控。

### 2.5 四个状态的代码行为

| 状态          | 进入或退出条件                                         | `pacing_rate` 的实际变化                                  |
| ------------- | ------------------------------------------------------ | --------------------------------------------------------- |
| `mccStart`    | 初始状态；RTT 超过阈值时进入 Stop                      | 每个被确认包增加 $M/R_0$ byte/s                           |
| `mccStop`     | $\bar R>gR_0$ 时进入；条件恢复后进入 Guard             | 进入时令 $x\leftarrow\max(\widehat D,128M/R_0)$，此后保持 |
| `mccGuard`    | Stop 条件解除，或 ProbeRTT 结束                        | 每四个 `OnPacketAcked` 回调增加 $M$ byte/s                |
| `mccProbeRTT` | 超过 5 秒没有刷新最小 RTT，且距上次 ProbeRTT 超过 5 秒 | 持续至少 200 ms；状态中不执行 Start/Guard 增长            |

ProbeRTT 退出时，如果进入前是 Start，则回到 Start；其他旧状态统一进入 Guard。

## 三、想法与相关工作定位

### 3.1 MCC 具备价值的部分

MCC 不是从抽象问题出发，而是从高 RTT、高随机丢包链路上的 HTTP/3 性能问题出发，并完成了 quic-go 接口级实现。它包含几个明确的工程选择：

- 不把丢包直接等同于拥塞，以适配存在随机丢包的路径；
- 使用最小 RTT 与当前 RTT 的差值提前观察排队；
- 用交付率更新发送尺度，而不是只维护传统 AIMD 窗口；
- 通过 ProbeRTT 更新基线；
- 试图在低延迟与对 buffer-filling 流的竞争力之间提供两种模式。

这些选择组成了一个可讨论、可实现、可实验的状态机，而不是只有概念描述。

### 3.2 哪些成分已有明确先例

MCC 的单个基础部件并不是空白领域：

- [TCP Vegas](https://www.cs.princeton.edu/courses/archive/fall06/cos561/papers/vegas.pdf) 已使用基础 RTT、期望吞吐和实际吞吐估计网络中的额外数据，并据此调整窗口。
- [LEDBAT RFC 6817](https://www.rfc-editor.org/rfc/rfc6817.html) 明确定义了以近期最小端到端时延估计 base delay，再以当前时延减去 base delay 估计排队时延。
- [BBR](https://www.ietf.org/archive/id/draft-ietf-ccwg-bbr-04.html) 使用 delivery rate、最小 RTT、pacing、inflight cap 和 ProbeRTT 建立路径模型。
- [Copa](https://www.usenix.org/conference/nsdi18/presentation/arun) 使用排队时延决定目标速率，并额外设计了与 buffer-filling 流竞争的模式切换。
- [Nimbus](https://arxiv.org/abs/1802.08730) 通过主动速率扰动及对交叉流响应的观测，识别交叉流性质；它也明确指出，仅靠瞬时排队量无法完成这种区分。

因此，MCC 当前更准确的创新定位是：**面向特定高随机丢包 WAN 的相对 RTT 阈值、交付率冻结和周期性重新基线的组合控制策略**。如果要把创新中心放在“测量本流的拥塞贡献”，则还需要加入能够建立因果关系的观测机制。

## 四、理论分析

### 4.1 `SRTT-MinRTT` 测到的是总排队时延，不是本流贡献

考虑单瓶颈流体模型。设瓶颈容量为 $C$，总队列为 $Q(t)$，第 $i$ 个流的发送速率为 $x_i(t)$：

$$
\dot Q(t)=
\begin{cases}
\sum_i x_i(t)-C, & Q(t)>0\\
\max\left(0,\sum_i x_i(t)-C\right), & Q(t)=0
\end{cases}
$$

对经过该瓶颈的流：

$$
R_i(t)=R_{0,i}+\frac{Q(t)}{C}
$$

于是 MCC 观察到的差值近似为：

$$
R_i(t)-R_{0,i}=\frac{Q(t)}{C}
$$

右侧是所有流共同形成的队列，而不是 $Q_i/C$。两个不同场景可以产生完全相同的 RTT 历史：

- 场景 A：主要由本流速率增长形成队列；
- 场景 B：本流速率不变，主要由交叉流形成队列。

仅看当前 RTT 差值无法区分这两个场景。因此，这个信号能够说明“路径上存在多少额外时延”，不能直接说明“这些时延中有多少由本流造成”。

若希望继续使用“贡献”一词，需要引入额外可识别信息，例如：

- 对本流速率施加幅度受控、均值不变的扰动，观察队列斜率或交付率响应；
- 使用 ECN、逐流队列、瓶颈遥测等网络反馈；
- 建立包含自身输入、交叉流和队列动态的状态估计器，并验证可识别性。

否则，建议将文档中的“拥塞贡献”改为“相对排队压力”或“额外 RTT 压力”。

### 4.2 阈值在队列模型中的真实含义

MCC 的 Stop 条件为：

$$
\bar R-R_0>(g-1)R_0
$$

代入 $Q/C$：

$$
Q>(g-1)CR_0
$$

而 $CR_0$ 正是基础路径对应的 BDP，所以该条件的含义是：

- 低延迟模式：总队列超过约 1 BDP 后触发；
- 普通模式：总队列超过约 2 BDP 后触发。

这一定义主要控制的是**允许形成多少总队列**，而不是公平份额。它可以解释带宽利用率与尾延迟之间的权衡，但不能由此直接推出某个流已经超过公平份额。

### 4.3 当前控制律没有给出等份额收敛机制

假设两个 MCC 流在同一瓶颈上，已有速率 $x_1\neq x_2$。

进入 Stop 时，每个流将自己的 `pacing_rate` 设置为各自当前交付率。若交付率本来不相等，这一步会保留已有差异。

在 Guard 中，源码每四个被确认包增加一个 MSS。若每次 ACK 回调平均确认 $k_i$ 个包，流 $i$ 的 ACK 回调频率近似为：

$$
f_i\approx\frac{y_i}{k_iM}
$$

其中 $y_i$ 是实际交付率。于是：

$$
\frac{dx_i}{dt}\approx\frac{y_i}{4k_i}
$$

当 $y_i$ 与 $x_i$ 近似成比例、ACK 策略相同时，这是一种近似乘性增长。两个流的速率比不会因此自动趋近 1：

$$
\frac{d}{dt}\ln\frac{x_1}{x_2}\approx0
$$

因此，“每个流守住自己的份额”可以描述设计愿望，但还没有证明算法会把任意初始份额变成相等份额。实际公平性可能来自丢包、ProbeRTT 不同步、ACK 行为和队列竞争等附加效应，这些需要实验测量，不能仅由当前状态机推出。

### 4.4 RTT 公平性需要单独检验

Start 中每个 ACK 的增量为：

$$
\Delta x=\frac{M}{R_0}
$$

因此不同基础 RTT 的流具有不同增长尺度。另一方面，阈值又与 $R_0$ 成比例，长 RTT 流允许更大的绝对排队时延。这两种 RTT 依赖会同时影响最终份额，不能从同 RTT 双流结果外推到异 RTT 公平性。

至少需要测试 $R_{0,1}:R_{0,2}$ 为 1:1、1:2、1:4、1:10 的组合，并分别交换先后启动顺序。

### 4.5 与 BBR/CUBIC 竞争的机制实质

面对持续填充缓冲区的流，传统 delay-based 算法容易因为排队时延升高而持续退让。MCC 的应对方式是：

1. Stop 中不按丢包直接乘性减小；
2. 一段时间后重新建立较高 RTT 基线；
3. 将原有 standing queue 的一部分吸收到新的 `backGroundRTT`；
4. 随后进入 Guard 继续增长。

这能够增强竞争性，但其含义是**接受已有 standing queue，并把它重新解释为路径基线的一部分**。它是一种“延迟控制与竞争性之间的策略选择”，不能证明自身对拥塞的贡献较小。基线污染还可能带来晚到流优势和持续队列，需要通过启动顺序实验验证。

### 4.6 随机丢包目标成立，但适用域应明确

目标路径存在 2%—10% 随机或非拥塞相关丢包，因此不根据单次丢包直接降低速率是有场景依据的。源码仍会通过 ACK 交付率间接感知有效交付能力下降。

但在浅缓冲、无随机丢包、丢包主要来自队列溢出的路径上，丢包本身包含拥塞信息。当前 `OnCongestionEvent()` 不修改速率或窗口，因此算法是否保持稳定和协议间公平，需要单独验证。适合将 MCC 的结论限定为目标 WAN 场景，直到完成更广泛矩阵测试。

## 五、源码与工程实现分析

### 5.1 优先级 P0：当前 pacing 并未按 `nextSendTime` 执行

[MCC 的 `HasPacingBudget()`](https://github.com/qiulaidongfeng/quic-go/blob/2947a072a3396baaf7843f49a7349a52774e470a/internal/congestion/mcc.go#L109-L121) 只比较：

$$
\widehat D<gx
$$

它不检查：

$$
now\geq nextSendTime
$$

quic-go 的调用顺序是先调用 `HasPacingBudget(now)`；只有返回 `false` 后，连接层才读取 `TimeUntilSend()` 并设置 pacing deadline。发送循环还会在同一个 `now` 值下连续构造多个包。结果是：

- 当 ACK 交付率低于门限时，可以连续发送一批包；
- `OnPacketSent()` 虽然计算了下一发送时间，但这个时间没有参与下一次 `HasPacingBudget()` 判断；
- 多个同批次包的 `sentTime` 相同，`nextSendTime` 也没有按包长累计债务；
- 实际行为更接近“ACK 交付率门控的突发发送”，而不是严格 pacing。

建议把“交付率估计器”和“pacer”分离。最直接的实现方案是复用同一包中的 quic-go `pacer`，或至少采用：

$$
nextSendTime\leftarrow
\max(nextSendTime,sentTime)+\frac{L}{rate}
$$

并令 `HasPacingBudget(now)` 检查时间预算，同时保留一个明确、有限的 burst allowance。

### 5.2 优先级 P0：初始 `smoothedRTT==0` 时交付率计算没有定义

在第一个有效 ACK 到来前：

$$
\bar R=0
$$

但 `update_lastbandwidth_filter()` 会计算：

$$
\widehat D=\frac{delivered}{0}
$$

该表达式先经过浮点除法，可能产生 `NaN` 或无穷，再转换为 `protocol.ByteCount`。其结果不应作为正常控制值。

建议显式规定启动语义，例如：

```go
if b.smoothedRTT <= 0 {
    return 0
}
```

并由真正的初始 cwnd/pacer 控制首个 RTT，而不是依赖非有限浮点值的整数转换结果。

### 5.3 优先级 P0：文档所称的 128 MSS 初始上限不是实际硬上限

`CanSend()` 在 $F<128M$ 时直接允许发送；超过该值后，如果 RTT 条件 `r` 为真，最终仍然返回 `true`。启动阶段 $\bar R=0$ 时 `r` 为真，因此 128 MSS 并不会形成硬窗口上限。

此时发送量最终可能由 socket/send queue、quic-go 的全局 outstanding packet 限制等外围条件约束，而不是由 MCC 的 128 MSS BDP 上限约束。

建议把在途量限制写成始终存在的明确上限，再在不同状态改变 gain；不要让 `r==true` 完全绕过窗口。`GetCongestionWindow()` 也应返回真正参与 `CanSend()` 的上限，避免日志显示值与实际限制不一致。

### 5.4 优先级 P0：MCC 重复实现 RTT 估计，并偏离 QUIC 的 RTT 语义

quic-go 已在 ACK 路径中按 QUIC 规则更新统一的 `RTTStats`：只选择适合的新 ACK 样本，并处理 ACK delay。随后，框架又对 ACK frame 中每个计入 bytes-in-flight 的包调用 MCC 的 `OnPacketAcked()`。

MCC 对每个这样的包重新计算：

$$
rtt=eventTime-sentTime[packetNumber]
$$

并以 1/4 权重更新 SRTT。这意味着：

- 一个 ACK frame 确认多个包时，同一 ACK 到达事件会产生多个 RTT 样本；
- 较早发送的包会产生更大的样本；
- 未扣除 QUIC ACK delay；
- ACK 聚合、重排序和 delayed ACK 会直接改变 MCC 的平滑 RTT。

[RFC 9002](https://www.rfc-editor.org/rfc/rfc9002.html) 与 quic-go 已有实现提供了统一 RTT 语义。建议让 MCC 直接读取或接收该 RTT 状态，而不是维护第二套 RTT 估计器。

### 5.5 优先级 P0：ECN 事件可能先删除待 ACK 包的时间戳

在 quic-go ACK 处理路径中，如果 ECN tracker 判断发生拥塞，会先调用：

```text
OnCongestionEvent(largestAcked, 0, priorInFlight)
```

随后才对这个 ACK 中的包调用 `OnPacketAcked()`。

MCC 的 `OnCongestionEvent()` 无条件删除 `sentTimes[number]`。因此，在 ECN CE 场景中，`largestAcked` 的发送时间可能先被删除，随后 `OnPacketAcked()` 从 map 取到零值，并计算出一个接近单调时钟起点的巨大 RTT 样本。

建议至少：

- 使用 `sentTime, ok := b.sentTimes[number]` 检查键是否存在；
- 当 `lostBytes==0` 表示 ECN 事件时，不把 ACKed packet 的时间戳当作丢包记录删除；
- 更彻底的方案仍是复用统一 RTTStats。

### 5.6 优先级 P1：`sentTimes` 的键和生命周期不覆盖 QUIC 语义

QUIC 的 Initial、Handshake 和 Application Data 是独立 packet number space，数值相同的 packet number 可以同时存在。MCC 的 map 只以 `PacketNumber` 为键，没有 packet number space，因此握手阶段可能发生时间戳覆盖。

此外，quic-go 会对非 ACK-eliciting 包调用 `OnPacketSent()`，但只对计入 bytes-in-flight 的包调用 MCC 的 `OnPacketAcked()`。MCC 当前不检查传入的 `isRetransmittable`/实际 `isAckEliciting`，所以 ACK-only 等包也会进入 `sentTimes`，却不一定被对应删除，形成持续增长的记录。

建议：

- 只记录确实会收到拥塞控制 ACK 回调的包；
- 使用包含 packet number space 的唯一键；
- 或取消这张表，直接使用框架已经维护的逐包信息和 RTTStats。

### 5.7 优先级 P1：`backGroundRTT` 不是严格的“最近 5 秒最小 RTT”

源码平时只要看到更小 RTT 就更新 `backGroundRTT`，并长期保留；超过 5 秒没有出现新低值后进入 ProbeRTT，把背景 RTT 直接重置为 5 秒，再由随后 ACK 降低它。

所以它实际上是：

> “保持历史低值，低值超过 5 秒未刷新后触发一次 200 ms 重新采样”

而不是：

> “对最近 5 秒所有 RTT 样本计算滑动最小值”

两者在 route change、standing queue、短时 RTT 波动下会产生不同结果。建议文档按实际语义改写，或者实现真正的时间窗口最小值过滤器。

### 5.8 优先级 P1：ProbeRTT 的“窗口减半”会受到基线重置影响

进入 ProbeRTT 时，代码先把 `backGroundRTT` 设为 5 秒。因为：

$$
W=xR_0
$$

这会在下一个较小 RTT 样本到来前暂时放大 `GetCongestionWindow()`。ProbeRTT 中的 `0.5` gain 作用于已经改变的 $R_0$，所以不等于“把进入 ProbeRTT 前的实际窗口减半”。128 MSS 的提前返回也可能覆盖 ProbeRTT 的更小限制。

建议保留旧 $R_0$ 和旧 inflight cap，先按明确目标排空，再用独立的 `probeMinRTT` 收集新样本，ProbeRTT 结束时才提交新基线。

### 5.9 优先级 P1：Guard 增长依赖 ACK 回调方式，不是严格每秒 1.25 倍

源码每四个 `OnPacketAcked()` 回调增加一个 MSS，没有直接使用经过时间，也没有按 `ackedBytes` 归一化。

在每个包分别进入回调、交付率近似等于 `pacing_rate` 时，它可以近似产生每秒 25% 左右的增长；但以下因素都会改变实际增长率：

- delayed ACK 或 ACK frequency；
- 一个 ACK frame 确认多少包；
- ACK 聚合与重排序；
- 包大小变化；
- 应用受限阶段。

建议直接按时间或已确认字节归一化，例如定义目标连续增长率，再用：

$$
\Delta x=\alpha x\Delta t
$$

或按每 RTT 一次的明确控制更新实现。这样“每秒 1.25 倍”才是协议参数，而不是 ACK 行为的副产物。

### 5.10 优先级 P1：交付率估计需要处理 ACK 压缩和应用受限

当前估计器使用“一个 SRTT 内 ACK 的总字节数 / SRTT”。它在稳定长流中可以提供简单平均值，但会受到以下影响：

- 刚启动或刚从 idle 恢复时窗口未填满，数值偏低；
- ACK compression 会短时偏高；
- 一个 RTT 没有 ACK 后会迅速变成零；
- 应用受限、接收端流控受限与网络瓶颈无法区分；
- 使用当前 SRTT 动态改变历史窗口边界。

BBR 的 delivery-rate sampling 专门保存 per-packet delivered/time 状态并标记 app-limited。MCC 可以复用已有实现思想，而不必把完整 BBR 状态机一起引入。

### 5.11 其他工程观察

- `expire_sum` 使用平台相关宽度的 `int`，而 ACK 字节和 `delivered` 使用 `int64` 类型的 `ByteCount`；可统一为 `protocol.ByteCount`。
- `ackinfo = ackinfo[keep_start_index:]` 长期保留底层数组，建议使用环形缓冲或周期性整理。
- `SetMaxDatagramSize()` 改变 MSS 后不会重新调整当前 pacing rate；需要明确 PMTU 变化时希望保持 byte/s 还是 packet/s 语义。
- `OnRetransmissionTimeout()` 在当前 quic-go 调用路径中主要由 path migration 路径触发，普通丢包调用的是 `OnCongestionEvent()`。函数名不等于实际常规丢包恢复路径。
- `InRecovery()` 把 Stop 暴露成 recovery，但 Stop 并非传统 loss recovery；这只影响调试语义，却可能误导 qlog/监控使用者。
- 源码没有 MCC 状态、RTT 阈值、实际发送率、inflight cap 的结构化日志，难以确认实测时真正由哪个门控条件限制发送。

## 六、文档叙述与代码行为对照

| 文档叙述                 | 代码实际行为                                                 | 结论                                                 |
| ------------------------ | ------------------------------------------------------------ | ---------------------------------------------------- |
| 最近 5 秒最小 RTT        | 低值长期保留；5 秒未刷新后触发 200 ms 重采样                 | 不是严格滑动窗口最小值                               |
| RTT 差值代表本流拥塞贡献 | 差值代表路径总额外 RTT                                       | 可作为拥塞压力，不能直接归因                         |
| 初始 BDP 上限为 128 MSS  | 小于 128 MSS 必放行；RTT 条件为真时超过后仍可放行            | 128 MSS 不是当前实现的硬上限                         |
| pacing 控制发送          | ACK 交付率决定 budget；deadline 不参与正常允许判断           | 当前不是严格 pacer                                   |
| Stop 保持当前交付速率    | `pacing_rate` 设为当前交付率，但 budget 阈值仍为 $g$ 倍，另受 inflight 限制 | “保持”依赖多个门控共同作用                           |
| Guard 每秒增长 1.25 倍   | 每四个被确认包增加一个 MSS                                   | 仅在特定 ACK/包长条件下近似成立                      |
| ProbeRTT 把 BDP 上限减半 | gain 为 0.5，但同时把 $R_0$ 重置为 5 秒，且存在 128 MSS 提前放行 | 不等于旧窗口严格减半                                 |
| 最多 5.2 秒后适应激进流  | 需同时满足两个 5 秒条件，并在调用路径到达时退出 200 ms ProbeRTT | “5.2 秒”是理想下界式描述，不是所有执行路径的严格上界 |

## 七、现有实验材料能够支持什么

文章提供了以下有价值的信息：

- 发送端、接收端和 AS；
- 操作系统；
- 约 160+ Mbps 瓶颈带宽；
- 最小 RTT 和通常 RTT；
- 夜间丢包范围；
- 对两种阈值模式的定性观察。

但公开材料中没有看到以下内容：

- 每组实验的原始日志或 qlog；
- MCC、BBRv1、NewReno/CUBIC 的同栈定量表格；
- 重复次数、时间段、均值/中位数/置信区间；
- 公平性计算公式、流数、启动顺序和统计窗口；
- 队列长度、AQM、瓶颈位置和 UDP 是否被策略限速；
- 对照实验脚本、对象大小、连接复用与否；
- 实际发送率与 `pacing_rate` 的吻合验证。

因此，目前可以支持的结论是：

> 作者在指定真实链路上观察到 MCC 原型具有继续研究的性能现象。

尚不能仅由现有材料支持：

> MCC 普遍优于 BBRv1、能够快速收敛到公平份额、能够与 BBR/CUBIC 近似公平，或能够测量本流拥塞贡献。

HTTP/2+TCP BBR 与 HTTP/3+QUIC NewReno 的直接比较还包含协议栈、TLS/握手、UDP 路径策略、GSO、实现和 CPU 等混杂因素。算法对照应固定为同一 quic-go/HTTP3 栈，只替换拥塞控制器。

## 八、测试与验证结果

在 Go 1.25.0 下对指定源码提交进行了测试：

- `go test ./internal/congestion`：通过；
- 仓库中没有 `mcc_test.go`，现有 congestion 测试没有直接实例化或验证 MCC；
- `go test ./...`：大部分包通过，但根包的 `TestConfigClone`、`TestConfigDefaultValues` 失败；另有两项 UDP buffer 测试因当前运行环境返回 `operation not permitted` 而失败。

其中，MCC 指定提交没有修改导致上述两项配置测试的文件，因此不能把全量测试失败全部归因于 MCC；但这也说明当前 fork 的持续集成状态没有达到全绿，MCC 本身更缺少专项行为测试。

建议至少增加以下确定性单元测试：

1. 第一个 ACK 前不得出现除零/非有限速率；
2. 给定假时钟和目标速率，1 秒实际发送字节误差在明确范围内；
3. 多包同一发送循环时，pacing debt 必须累计；
4. 1、2、10 个包每 ACK 时，Guard 的时间增长率保持一致；
5. Initial/Handshake/1-RTT packet number 重号不污染 RTT；
6. ACK-only 包长期发送时 `sentTimes` 大小有界；
7. ECN CE ACK 不产生零时间戳 RTT；
8. ProbeRTT 期间实际 inflight 达到设计目标；
9. route change 后背景 RTT 能上调和下调；
10. 每个状态的进入、退出、速率和窗口变化与形式化状态表一致。

## 九、建议的最小改进路线

### 第一阶段：先让代码精确实现文档中的算法

按优先级建议：

1. 使用真正的时间/预算 pacer，交付率估计不再承担 pacing budget 的职责；
2. 统一使用 quic-go RTTStats，移除当前 packet-number-only 的发送时间表；
3. 把初始 cwnd、正常 cwnd、Stop cap 和 ProbeRTT cap 写成始终有效的明确公式；
4. 修复零 RTT、ECN 删除时间戳和非 ACK-eliciting 包记录问题；
5. 将 Guard 增长改成按时间或按 ACK 字节归一化；
6. 明确实现“真正 5 秒滑动最小值”或修改文档名称；
7. 增加状态和控制量日志及专项单元测试。

完成这一步后，实测才是在验证同一个、可精确定义的 MCC。

### 第二阶段：缩小并强化理论主张

建议先把核心命题改为：

> MCC 使用相对排队时延判断路径排队压力，并通过交付率冻结与周期性重新基线，在高随机丢包 WAN 上平衡利用率、延迟和对 buffer-filling 流的竞争性。

这个命题与当前机制一致，也可以直接实验。

如果仍希望证明公平性，需要明确：

- 公平定义：同 RTT 流公平、RTT 异质公平，还是与 CUBIC/BBR 的协议间友好性；
- 平衡点；
- 每个状态下的连续或离散更新方程；
- 从任意初始速率是否收敛；
- ACK ratio、随机丢包和 ProbeRTT 对平衡点的影响。

如果希望保留“贡献测量”，则建议增加受控微扰：在不改变一个周期平均速率的前提下交替使用 $x(1+\epsilon)$ 和 $x(1-\epsilon)$，再比较队列斜率、交付率或交叉流响应。只有观测到输出随本流输入变化，才能建立比静态 RTT 差值更强的因果解释。

### 第三阶段：建立可重复实验矩阵

建议统一使用相同 HTTP/3/quic-go 代码，只替换拥塞控制器，并至少覆盖：

| 维度     | 建议取值                                                     |
| -------- | ------------------------------------------------------------ |
| 算法     | MCC 两模式、NewReno、CUBIC、作者 BBRv1；条件允许时加入较新 BBR |
| 随机丢包 | 0%、0.1%、1%、2%、5%、10%                                    |
| 基础 RTT | 10、50、100、240 ms                                          |
| 缓冲区   | 0.5、1、2、5、10 BDP                                         |
| 队列     | DropTail、fq_codel/CoDel、可用时加入 ECN                     |
| MCC 流数 | 1、2、4、8                                                   |
| RTT 比   | 1:1、1:2、1:4、1:10                                          |
| 事件     | 新流加入、旧流退出、容量阶跃、RTT/路由变化、应用 idle 后恢复 |
| 竞争流   | MCC、NewReno/CUBIC、BBR、恒定码率流                          |

每组报告：

- goodput 和 wire throughput；
- 平均、P95、P99 RTT 与排队时延；
- 丢包率和重传字节；
- Jain 指数；
- 到达公平份额 ±10% 的时间；
- 1 ms、10 ms、1 RTT 窗口内的发送突发性；
- CPU、内存和 `sentTimes`/ACK 历史长度；
- 状态驻留时间与 ProbeRTT 次数。

真实 AS20473→AS17816 路径测试可作为最后一组外部有效性实验，并采用多晚、相同时间段、随机化算法顺序的重复测量。

## 十、整体评价

MCC 目前最准确的定位是：

> 一个由真实跨域 HTTP/3 性能问题驱动、已经接入 quic-go、以相对 RTT 和交付率为核心信号的实验性混合拥塞控制原型。

它的工程动机明确，状态机也体现了作者对“随机丢包、延迟控制、带宽利用率、对激进流竞争性”之间矛盾的实际思考。作为社区交流稿，它有足够的讨论价值。

但从理论上看，`SRTT-MinRTT` 只能观测总排队压力，不能完成本流归因；当前增长和冻结机制也没有建立等份额收敛。工程上，pacing、初始 inflight、RTT 采样、ECN、ProbeRTT 和 ACK 依赖增长等问题会使实际执行偏离文档算法。公开实验又暂时缺少足以排除混杂因素的数据。

因此，当前阶段可以评价为：

- **问题与想法：成立，值得继续；**
- **“贡献测量”理论：尚未成立，应改名或增加因果探测；**
- **公平性：存在设计目标和经验假设，尚无理论或充分实验支持；**
- **源码：可编译的研究原型，但关键控制路径需要修正；**
- **实验结论：属于初步真实链路观察，尚未形成可复现证据链。**

最有效的下一步不是继续增加状态或阈值，而是先完成真正 pacing、统一 RTT 语义和 MCC 专项测试，然后用两个同 RTT MCC 流、一个 MCC 对一个 BBR/CUBIC 流这两组最小实验，逐条验证公平性和竞争性主张。

## 十一、建议交流时向作者确认的问题

1. 作者希望“贡献”表示总排队压力，还是严格表示本流造成的因果份额？
2. `pacing_rate` 是目标交付率，还是实际线速？为什么 `HasPacingBudget` 和包间隔都再乘 $g$？
3. 当前实测是否验证过实际逐毫秒发送字节与设定 pacing rate 一致？
4. “Guard 每秒 1.25 倍”是在什么 ACK frequency、包长和 ACK 聚合条件下测得的？
5. 公平性使用什么定义、流数、启动顺序、统计区间和 Jain 指数？
6. 多条 MCC 流初始速率不相等时，代码中的哪一步负责缩小速率差？
7. 异 RTT 流竞争时，作者预期短 RTT 流还是长 RTT 流占优？
8. ProbeRTT 进入时把 `backGroundRTT` 设为 5 秒，是哨兵值还是希望立即参与 cwnd 计算？
9. 对 BBR/CUBIC 的比较是否在同一个 HTTP/3/quic-go 栈内，只替换拥塞控制器？
10. 是否可以公开一次完整 qlog、运行参数、原始吞吐/RTT 序列和复现实验命令？

这些问题不预设算法结论，能够直接帮助把设计直觉转化为可验证的协议定义。

## 参考资料

- Lawrence S. Brakmo, Larry L. Peterson, [TCP Vegas: End to End Congestion Avoidance on a Global Internet](https://www.cs.princeton.edu/courses/archive/fall06/cos561/papers/vegas.pdf)
- Neal Cardwell et al., [BBR Congestion Control, IETF Internet-Draft](https://www.ietf.org/archive/id/draft-ietf-ccwg-bbr-04.html)
- Venkat Arun, Hari Balakrishnan, [Copa: Practical Delay-Based Congestion Control for the Internet](https://www.usenix.org/conference/nsdi18/presentation/arun)
- Prateesh Goyal et al., [Elasticity Detection: A Building Block for Internet Congestion Control](https://arxiv.org/abs/1802.08730)
- IETF, [RFC 6817: Low Extra Delay Background Transport](https://www.rfc-editor.org/rfc/rfc6817.html)
- IETF, [RFC 9002: QUIC Loss Detection and Congestion Control](https://www.rfc-editor.org/rfc/rfc9002.html)





