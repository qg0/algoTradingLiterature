
根据worldquant发表的论文《101 Formulaic Alphas 》 ，其中公式化地给出了101个alpha因子。论文地址在这：https://arxiv.org/pdf/1601.00991.pdf  他们根据数据挖掘的方法发掘了101个alpha，据说里面80%的因子仍然还行之有效并运行在他们的production中。
对于worldquant的websim回测系统进行了模拟，可改变alpha表达式自行编写expression形式的alpha进行回测。如：alpha=1/close.

 

         对其中的函数进行如下解释：

可分为横截面函数和时间序列函数两大类，其中时间序列函数名多为以ts_开头
大部分函数命名方式较为直观
abs(x) log(x)分别表示x的绝对值和x的自然对数
rank(x)表示某股票x值在横截面上的升序排名序号，并将排名归一到[0,1]的闭区间
delay(x,d)表示x值在d天前的值
delta(x,d)表示x值的最新值减去x值在d天前的值
correlation(x,y,d) covariance(x,y,d)分别表示x和y在长度为d的时间窗口上的Pearson相关系数和协方差
ts_min(x,d) ts_max(x,d) ts_argmax(x,d) ts_argmin(x,d) ts_rank(x) sum(x,d) stddev(x,d)等均可以通过函数名称了解其作用
具体函数详情见Available Operators                                                                               
    alpha值意义：在每个迭代或者说在每一个交易日，通过计算表达式得到的 每一只股票的Alpha 值，可以得出进行买卖的工具（股票）数量。Alpha  不是表示想要花多少钱买卖，而是在当日的资金比例。   
    Neutralization（中性化）： 中性化是一种将我们的策略按照市场、行业或子行业进行中性化的操作。当设置 Neutralization=“market” （中性化=整个市场）时，它会进行以下操作：
alpha = alpha – mean(alpha)

    基本上，它使 Alpha 向量的均值为零，使得市场上的“净”头寸为零。换句话说，多头头寸和空头头寸相互完全抵消，使我们的策略完全市场中性化。

    当设置 Neutralization=“industry”或“sub-industry”（中性化=行业或子行业），那么所有在 Alpha 向量中的金融工具将会按照行业或者子行业分成数个小组，然后对每个小组进行中性化。

    if i = **Subindustry 中性化：将i这只股票的alpha值减去子行业alpha的均值

    alpha(i) = alpha(i) – mean((alpha.subindustry(i)))

    中性化（Neutralization）指的是将原始的Alpha值放入不同组，然后在各组内进行标准化（Normalization，指用每个数值减去平均值）的一种操作。 一个组可以是整个市场。 或者根据其他分类来定义组，例如行业或者子行业（基于申银万国行业）。 这样做是为了避免收益受所选组的走向影响，而只注重于股票未来的相对收益。 中性化操作之后，整个投资组合处于中性仓位（多头和空头各占一半）。 这种做法可防止投资组合受市场剧烈波动的影响，还能过滤一些虚假信号。

这里有一个简单的例子，假设 Alpha = close（收盘价格）。
假设您的样本空间里面有 5 只股票（A、B、C、D、E），而在某个特定的日期（20100104）它们的收盘价分别如下（以美元计）：

 

 

金融工具（股票）：  A    B    C    D    E

收盘价：             6    5     2    8     4

现在，您想要使用这些收盘价来计算下一个交易日的 Alpha 权重。 您首先要做的就是写出 Alpha 表达式，在这里即 Alpha = close。因此您首先要有一个构建一个“close”（收盘价）的向量，即(6, 5, 2, 8, 4)。【注：如果您的表达式是 Alpha = 1 / close，那么您需要构建的向量也必须是“1 / close”，即(1/6, 1/5, 1/2, 1/8, 1/4) = (0.167, 0.2, 0.5, 0.125, 0.25)。】



现在您有了向量（6，5，2，8，4），但这不并是权重向量。权重的向量需要经过标准化处理到1。因此我们将每个元素除以它们的总和（= 25），得到的元素总和等于 1。
于是得到了新的向量：(6 / 25, 5 / 25, 2 / 25, 8 / 25, 4 / 25) = (0.24, 0.20, 0.08, 0.32, 0.16)。此向量的总和是 1。这就是我们的投资组合。我们将每个元素再乘以资金规模（2000 万），就能得到我们需要在每只股票上投资的金额。



这是我们将 Neutralization（中性化）设定为“None”得到的结果。不过这种方法会让我们的策略受到市场风险的影响。于是，我们设置 Neutralization = "Market"（中性化 = “市场”）。在这种情况下，我们首先再一次构建一个“close”（收盘价）的向量 = (6, 5, 2, 8, 4)。
现在我们使其 "mean-neutral （均值中性化）"，也就是将每个元素减去他们的均值（=5），使得向量的总和为 0。
于是得到了均值中性化的向量： (1, 0, -3, 3, -1)。您会发现此时所有元素的总和为零。


现在，若要对其进行标准化处理，我们忽略数值的正负，使元素总和为 1。首先我们计算元素绝对值的总和（= 1 + 0 + 3 + 3 + 1 = 8）。现在我们的将每个元素除以它们的总和，得到 (1 / 8, 0 / 8, -3 / 8, 3 / 8, -1 / 8) = (+0.125, 0, -0.375, +0.375, -0.125)。
现在，这就是标准化均值中性的权重向量。我们把它乘以资金规模 2000 万美元，就得到了我们在每只股票上需要投资的金额，如果数值为正，则表明对其建立多头头寸，如果数值为负，则表明对其建立空头头寸。此外，因为所有的正值相加等于 0.5，而负值相加等于 -0.5，这表明我们在多头头寸上投资了 1000 万美元，在空头头寸上也投资了 1000 万美元，满足了中性化策略的要求。