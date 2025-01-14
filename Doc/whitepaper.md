> 文章推荐：https://mirror.xyz/adshao.eth/qmzSfrOB8s6_-s1AsflYNqEkTynShdpBE0EliqjGC1U

## Abstract

这篇技术白皮书解释了Uniswap v2核心合约背后的一些设计决策。它涵盖了合约的新功能，包括ERC20之间任意交易对、一个硬化价格预言机，允许其他合约估算给定时间间隔内的加权平均价格，“闪电交换”允许交易者在稍后的交易中先收到资产并在其他地方使用它们，然后再支付费用，并且未来可以开启协议费。它还重新架构了合同以减少攻击面。本白皮书描述了Uniswap v2“核心”合同的机制，包括存储流动性提供者资金的配对合同和用于实例化配对合同的工厂合同。

## Introduction

Uniswap v1是以太坊区块链上的智能合约的链上系统，实现了基于“恒定产品公式”的自动化流动性协议。每个Uniswap v1交易对存储两种资产的汇集准备金，并为这两种资产提供流动性，保持准则不变：准备金乘积不能下降。交易者在交易中支付30个基点的费用，该费用归流动性提供者所有。这些合合约无法升级。

Uniswap v2是基于相同公式的新实现，具有几个新的高度可取特性。最重要的是，它使得可以创建任意ERC20/ERC20交易对，而不仅仅支持ERC20和ETH之间的交易对。它还提供了一个硬化价格预言机，在每个区块开始时累积两种资产的相对价格。这允许以太坊上的其他合约估算两种资产在任意时间间隔内加权平均价格。最后，它启用“闪电交换”，用户可以自由接收资产并将其用于链上其他地方，在事务结束时只支付（或返回）那些资产。

尽管合约通常无法升级，但有一个私钥可以更新工厂合约上的变量，以在交易中开启链上5个基点的费用。该费用最初将关闭，但未来可能会打开，在此之后，流动性提供者每笔交易将获得25个基点的收益，而不是30个基点。

如第3节所讨论的那样，Uniswap v2还修复了Uniswap v1的一些小问题，并重新设计了实现方式，减少了Uniswap的攻击面，并通过将“核心”合约中保存流动性提供者资金的逻辑最小化来使系统更易于升级。

本文介绍了核心合约的机制，以及用于实例化这些合约的工厂合约。要使用Uniswap v2，需要通过“路由”合约调用交易对合约，计算交易或存款金额并将资金转移到交易对合约中。

## New features

### ERC-20 pair

Uniswap v1使用ETH作为桥梁货币。每个交易对都包括ETH作为其资产之一。这使得路由更简单 - 每个ABC和XYZ之间的交易都通过ETH / ABC对和ETH / XYZ对进行，并减少了流动性的碎片化。

然而，这个规则对流动性提供者施加了重大成本。所有的流动性提供者都有ETH的风险敞口，并根据其他资产相对于ETH价格的变化遭受不可避免的损失。当两种资产ABC和XYZ相关联时——例如，如果它们都是美元稳定币——Uniswap交易对ABC/XYZ上的流动性提供者通常会比ABC/ETH或XYZ/ETH交易对承受更少的不可避免损失。

将ETH作为强制桥梁货币也会对交易者造成成本。交易者需要支付两倍于直接ABC/XYZ配对的费用，并且他们遭受了两倍的滑点。
Uniswap v2允许流动性提供商为任何两个ERC-20创建配对合约。
任意ERC-20之间的大量配对可能使寻找特定配对的最佳路径略微困难，但路由可以在更高层次上处理（可以是离线或通过链上路由或聚合器）。

> 可以创建任意ERC20/ERC20交易对，而V1只能创建ETH/ERC20交易对。（使用ETH作为必备的桥接货币会对流动性提供者和交易者造成成本和影响）
>
> 这样在交易对内部对token就可以统一处理，不再区分是eth还是erc20。为了支持ERC20/ERC20交易对，ETH相应的变成了WETH。

### Price oracle 

在时间t，Uniswap提供的边际价格（不包括费用）可以通过将资产a的储备除以资产b的储备来计算。

由于套利者会在Uniswap上进行交易，如果价格不正确（足以弥补费用），Uniswap提供的价格往往跟踪资产的相对市场价格。这意味着它可以用作近似价格预言机。

然而，Uniswap v1不安全用作链上价格预言机，因为它很容易被操纵。假设其他合约使用当前的ETH-DAI价格来结算衍生品。希望操纵测量价格的攻击者可以从ETH-DAI对中购买ETH，在衍生品合约上触发结算（导致其基于夸大的价格结算），然后将ETH卖回到该对中以按真实价格交易回去。这甚至可能是原子交易或由控制块内事务排序的矿工完成的。

Uniswap v2通过在每个区块的第一笔交易之前测量和记录价格来改进此预言机功能。这个价格比区块期间的价格更难操纵。如果攻击者提交一笔试图在区块结束时操纵价格的交易，其他套利者可能能够在同一个区块中立即提交另一笔交易进行回购。矿工（或使用足够 gas 填满整个区块的攻击者）可以在一个区块结束时操纵价格，但除非他们也挖掘下一个区块，否则他们可能没有特定优势来进行套利回购。

具体来说，Uniswap v2通过跟踪每个与合约交互的区块开始时价格的累积和来累积这个价格。每个价格都根据自上次更新它的区块时间戳以及经过的时间量进行加权。这意味着在任何给定时间（更新后），累加器值应该是合约历史上每秒钟点价值的总和。

为了估算从时间t1到t2的加权平均价格，外部调用者可以在t1时检查累加器的值，然后再在t2时检查一次，将第一个值减去第二个值，并除以经过的秒数。 （请注意，合约本身不会存储此累加器的历史值-调用者必须在该期间开始时调用合约以读取和存储此值。）

Oracle的用户可以选择何时开始和结束此期间。选择较长的时间段会使攻击者更难以操纵TWAP，但这会导致价格不够实时。

一个复杂的问题：我们应该用资产B来衡量资产A的价格，还是用资产A来衡量资产B的价格？虽然以B为单位计算出来的A现货价格总是等于以A为单位计算出来的B现货价格的倒数，但在特定时间段内，以B为单位计算出来的平均A价格并不等于以A为单位计算出来的平均B价格。例如，如果USD/ETH汇率在区块1中为100，在区块2中为300，则平均USD/ETH汇率将是200 USD/ETH，但平均ETH/USD汇率将是1/150 ETH/USD。由于合约无法知道用户想要使用哪种资产作为账户单位，Uniswap v2跟踪这两个价格。

另一个复杂性在于，有可能有人向该交易对合约发送资产——从而改变其余额和保证金价格——而不与之互动，因此也不会触发 Oracle 更新。如果合约仅检查自己的余额并根据当前价格更新 Oracle，则攻击者可以通过在块中第一次调用它之前立即将资产发送到合约来操纵 Oracle。如果最后一笔交易是在时间戳为 X 秒钟前的块中进行的，则合约将错误地将新价格乘以 X 然后累加它，尽管没有人有机会以那个价格进行交易。为了防止这种情况发生，核心合同在每次互动后缓存其储备，并使用从缓存储备派生出的价格而不是当前储备来更新 Oracle。除了保护Oracle免受操纵外，这种更改还使得本文3.2节所述的合同重构成为可能。

#### 精度

Solidity没有一等的非整型数的数据结构的支持，Uniswap v2用简单的二进制定点数格式编码和控制价格。具体来说，某一时间的价格存储为UQ112.112格式，意思是在小数点的任意一边都有112位精度，无符号。

选择UQ112.112格式是由于实用的原因，因为这些数可以被存在uint224中，在256位中剩余的32位空余。储备资金各自存在uint112中，剩余32位存储空间。这些空闲空间被用于之前描述的累加过程。具体来说，储备资金和时间戳存储在至少有一个交易的最近的区块中，mod 232（译者注：取余数）之后可以存进32位空间。另外，虽然任意时间的价格（UQ112.112数字）确保可以储存进224位中，但某段时间的累加值确保能存下。存储A/B和B/A累加价格空间尾部附加的32位用来存连续累加溢出的位。这样设计意味着价格预言只在每一个区块的第一次交易中增加了3次SSTORE操作（目前花费15000gas）。

### Flash Swaps

Uniswap v1中，用户用XYZ买ABC需要发送XYZ到合约，然后才能收到ABC。如果用户需要ABC为了获取他们支付的XYZ，这种方式不方便的。举个例子，用户可能在其他合约中用ABC买XYZ，为了对冲Uniswap上的价格，或者他们可能在Maker或Compound上卖出抵押品平仓然后返还给Uniswap。

**Uniswap v2添加了一个新的特性，允许用户在支付前接收和使用资产，只要他们在同一个交易中完成支付**。swap函数调用一个可选的用户指定的回调合约，在这之间转出用户请求的代币并且强制确保不变。一旦回调完成，合约检查新余额并且确保满足不变（在经过支付手续费调整后）。如果合约没有足够的资金，它会回滚整个交易。

用户也可以用同样的代币返还给Uniswap资金池而不完成互换。这高效地让任何人从Uniswap资金池中快速借取任何资产（Uniswap收取同样的千分之三的交易手续费）。

### Protocol fee

Uniswap v2包括0.05%协议手续费，可以打开或关闭，如果打开，手续费会被发送给工厂合约中指定的feeTo地址。

初始时，feeTo没有被设定，不收手续费。一个预先指定的地址feeToSetter可以在Uniswap v2工厂合约上调用setFeeTo函数，设置feeTo地址。feeToSetter也可以自己调用setFeeToSetter修改feeToSetter地址。

如果feeTo地址被设置了，协议将开始收取5个基点（0.05%）的手续费，也就是流动性提供者收取的30个基点（0.30%）手续费中的1/6将分配给协议。这意味着交易者将继续为每一笔交易支付0.30%的交易手续费，83.3%（5/6）的手续费（整笔交易的0.25%）将分配给流动性提供者，剩余的16.6%（手续费的1/6，整笔交易的0.05%）将分配给feeTo地址。

总共收集的手续费可以用自从上次手续费收集以来（译者注：k是常数乘积，可以看v1白皮书）的增长来计算。



















