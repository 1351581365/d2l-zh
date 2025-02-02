# 参数服务器
:label:`sec_parameterserver`

当我们从一个 GPU 迁移到多个 GPU 时，以及再迁移到包含多个 GPU 的多个服务器时（可能所有服务器的分布跨越了多个机架和多个网络交换机），分布式并行训练算法也需要变得更加复杂。通过细节可以知道，一方面是不同的互连方式具有区别极大的带宽（例如，NVLink 可以通过设置实现跨 $6$ 条链路高达 100 GB/s 的带宽，16 通道的 PCIe 4.0 提供 32 GB/s 的带宽，而即使是高速 100 GbE 以太网也只能提供10 GB/s）；另一方面是期望开发者既能完成统计学习建模还精通网络和系统也是不切实际的。

参数服务器的核心思想最先是由 :cite:`Smola.Narayanamurthy.2010` 在分布式隐变量模型的背景下引入的。然后，在 :cite:`Ahmed.Aly.Gonzalez.ea.2012` 中描述了 Push 和 Pull 的语义，在 :cite:`Li.Andersen.Park.ea.2014` 中描述了系统和开源库。下面，我们将介绍用于提高计算效率的组件。

## 数据并行训练

让我们回顾一下在分布式架构中数据并行训练的方法，因为在实践中它的实现相对简单，因此本节将排除其他内容只对其进行介绍。由于当今的 GPU 拥有大量的显存，因此在实际场景中（不包括图深度学习）只有数据并行训练这种并行策略值得推荐。图  :numref:`fig_parameterserver` 描述了在 :numref:`sec_multi_gpu` 中实现的数据并行的变体。其中的关键是参数在更新前需要在 GPU 0 上先进行梯度聚合，然后再重新广播给所有的 GPU。

![左图：单GPU训练。右图：多GPU训练的一个变体：（1）计算损失和梯度，（2）所有梯度聚合在一个GPU上，（3）发生参数更新，并将参数重新广播给所有GPU。](../img/ps.svg)
:label:`fig_parameterserver`

回顾来看，在 GPU 0 上进行聚合似乎是很随意的决定，当然也可以在 CPU 上聚合，事实上只要优化算法支持，在实际操作中甚至可以在某个 GPU 上聚合其中一些参数，而在另一个 GPU 上聚合另一些参数。例如，如果有四个与参数向量相关的梯度 $\mathbf{g}_1, \ldots, \mathbf{g}_4$，则可以一个 GPU 对一个 $\mathbf{g}_i (i = 1, \ldots, 4$) 地进行梯度聚合。

下面的推断似乎是轻率和武断的，毕竟数学应该是逻辑自洽的。但是，我们处理的是如 :numref:`sec_hardware` 中所述的真实的物理硬件，其中不同的总线具有不同的带宽。考虑一个如 :numref:`sec_hardware` 中所述的真实的 $4$ 路 GPU 服务器。如果它的连接是特别完整的，那么可能拥有一个 100 GbE 的网卡。更典型的数字在 1-10GbE 范围内，其有效带宽为 100 MB/s 到 1 GB/s。因为 CPU 的 PCIe 通道太少（例如，消费级的 Intel CPU 有 $24$ 个通道），所以无法直接与所有的 GPU 相连接，因此需要 [multiplexer](https://www.broadcom.com/products/pcie-switches-bridges/pcie-switches)。CPU 在 16x Gen3 链路上的带宽为 16 Gb/s，这也是每个 GPU 连接到交换机的速度，这意味着 GPU 设备之间的通信更有效。

![一个4路GPU服务器](../img/bw-hierarchy.svg)
:label:`fig_bw_hierarchy`

为了便于讨论，我们假设所有梯度共需 160 MB。在这种情况下，将其中 $3$ 个 GPU 的梯度发送到第 $4$ 个 GPU 上需要 $30$ 毫秒（每次传输需要 $10$ 毫秒 = 160 MB/16 GB/s）。再加上 $30$ 毫秒将权重向量传输回来，得到的结果是总共需要 $60$ 毫秒。如果将所有的数据发送到 CPU，总共需要 $80$ 毫秒，其中将有 $40$ 毫秒的惩罚，因为 $4$ 个 GPU 每个都需要将数据发送到 CPU。最后，假设能够将梯度分为 $4$ 个部分，每个部分为 $40$ MB，现在可以在不同的 GPU 上同时聚合每个部分。因为 PCIe 交换机在所有链路之间提供全带宽操作，所以传输需要 $2.5\times 3=7.5$ 毫秒，而不是 $30$ 毫秒，因此同步操作总共需要 $15$ 毫秒。简而言之，一样的参数同步操作基于不同的策略时间可能在 $15$ 毫秒到 $80$ 毫秒之间。 :numref:`fig_ps_distributed` 描述了交换参数的不同策略。

![参数同步策略](../img/ps-distributed.svg)
:label:`fig_ps_distributed`

请注意，我们还可以使用另一个工具来改善性能：在深度网络中，从顶部到底部计算所有梯度需要一些时间，因此即使还在忙着为某些参数计算梯度时，就可以开始为准备好的参数同步梯度了。想了解详细信息可以参见 :cite:`Sergeev.Del-Balso.2018` ，想知道如何操作可参考 [Horovod](https://github.com/horovod/horovod)。

## 环同步（Ring Synchronization）

当谈及现代的深度学习硬件上的同步时，我们经常会遇到大量定制的网络连接。例如，AWS p3.16xlarge 和 NVIDIA DGX-2 实例都使用到了 :numref:`fig_nvlink` 的连接结构。每个 GPU 通过 PCIe 链路连接到主机 CPU，该链路最多只能以 16 GB/s 的速度运行。此外，每个 GPU 还具有 $6$ 个 NVLink 连接，每个 NVLink 连接都能够以 300 Gbit/s 进行双向传输。这相当于每个方向每个链路约 18 GB/s。简言之，聚合的 NVLink 带宽明显高于 PCIe 带宽。问题是如何最有效地使用它。

![在 8 台 V100 GPU 服务器上连接 NVLink（图片由英伟达提供）](../img/nvlink.svg)
:label:`fig_nvlink`

结果表明，最优的同步策略是将网络分解成两个环，并用它们直接同步数据 :cite:`Wang.Li.Liberty.ea.2018` 。 :numref:`fig_nvlink_twoloop` 说明了网络可以分解为一个具有双 NVLink 带宽的环（1-2-3-4-5-6-7-8-1）和一个具有常规带宽的环（1-4-6-3-5-8-2-7-1）。在这种情况下，设计一个高效的同步协议是非常重要的。

![将 NVLink 网络分解为两个环。](../img/nvlink-twoloop.svg)
:label:`fig_nvlink_twoloop`

考虑下面的想法：给定一个环，由 $n$ 个计算节点（或GPU）组成，梯度可以从第一个节点发送到第二个节点。在第二个结点将本地的梯度与传送的梯度相加并发送到第三个节点，依此类推。在 $n-1$ 步之后，可以在最后访问的节点中找到聚合梯度。也就是说，聚合梯度的时间随节点数线性增长。但如果照此操作，算法是相当低效的。归根结底，在任何时候都只有一个节点在通信。如果我们将梯度分为 $n$ 个块，并从节点 $i$ 开始同步块 $i$，会怎么样？因为每个块的大小是 $1/n$，所以总时间现在是  $(n-1)/n \approx 1$。换句话说，当我们增大环的大小时，聚合梯度所花费的时间不会增加。这是一个相当惊人的结果。 :numref:`fig_ringsync` 说明了$n=4$个节点上的步骤顺序。

![跨4个节点的环同步。每个节点开始向其左邻居发送部分梯度，直到在其右邻居中找到聚合的梯度。](../img/ringsync.svg)
:label:`fig_ringsync`

如果我们使用相同的例子，跨 $8$ 个 V100 GPU 同步 160 MB，我们得到的结果大约是 $2 \cdot 160 \mathrm{MB} / (3 \cdot 18 \mathrm{GB/s}) \approx 6 \mathrm{ms}$。这比使用 PCIe 总线要好，尽管我们现在使用的是 8 GPU。请注意，这些数字在实践中要更糟一些，因为深度学习框架通常无法将通信组合成大的突发传输。

请注意，有一种常见的误解认为环同步与其他同步算法在本质上是不同的。与简单的树算法相比唯一的区别是同步路径稍微精细一些。

## 多机训练

在多台机器上进行分布式训练相对前面又增加了一个挑战：我们需要与服务器通信，而这些服务器只通过相对较低的带宽结构连接，这种连接的速度在某些情况下可能会慢一个数量级，因此跨设备的同步很棘手。毕竟，在不同机器上运行训练代码的速度会有细微的差别，因此如果我们想使用分布式优化的同步算法就需要 *同步*（synchronize）这些机器。:numref:`fig_ps_multimachine`说明了分布式并行训练是如何发生的。

1. 在每台机器上读取一组（不同的）批量数据，在多个 GPU 之间分割数据并传输到 GPU 的显存中。基于每个 GPU 上的批量数据分别计算预测和梯度。
2. 来自一台机器上的所有的本地 GPU 的梯度聚合在一个 GPU 上（或者它的一部分聚合在不同的 GPU 上）。
3. 每台机器的梯度被发送到其本地 CPU 中。
4. 所有的 CPU 将梯度发送到中央参数服务器中，由该服务器聚合所有梯度。
5. 然后使用聚合后的梯度来更新参数，并将更新后的参数广播回各个 CPU 中。
6. 每台机器的 CPU 将信息被发送到本地一个（或多个）GPU 中。
7. 所有 GPU 上的参数完成了更新。

![多机多 GPU 分布式并行训练。](../img/ps-multimachine.svg)
:label:`fig_ps_multimachine`

以上这些操作似乎都相当简单。实际上，它们可以在一台机器内高效地执行，但是当我们考虑多台机器时，就会发现中央的参数服务器成为了瓶颈。毕竟，每个服务器的带宽是有限的，因此对于 $m$ 个工作节点来说，将所有梯度发送到服务器所需的时间是 $\mathcal{O}(m)$。我们也可以通过将参数服务器数量增加到 $n$ 来突破这一障碍。此时，每个服务器只需要存储 $\mathcal{O}(1/n)$ 个参数，因此更新和优化的总时间变为 $\mathcal{O}(m/n)$。匹配这两个数字会产生稳定的伸缩性，而不管我们要处理多少工作节点。实际上，我们使用同一台机器作为工作节点和服务器。设计说明参考 :numref:`fig_ps_multips`（技术细节参考 :cite:`Li.Andersen.Park.ea.2014` ）。特别是，确保多台机器在没有不合理延迟的情况下工作并不容易。我们在下面忽略了关于阻塞的细节，只简单介绍一下同步和异步更新。

![上图：单参数服务器是一个瓶颈，因为它的带宽是有限的。下图：多参数服务器使用聚合带宽存储部分参数。](../img/ps-multips.svg)
:label:`fig_ps_multips`

## 键值存储

在实践中实现分布式多 GPU 训练所需的步骤绝非易事。这就是为什么使用一个公共抽象是值得的，即重新定义具有更新语义的 *键－值存储*（key-value store）的抽象。

在许多工作节点和许多 GPU 中，梯度 $i$ 的计算可以定义为

$$\mathbf{g}_{i} = \sum_{k \in \text{workers}} \sum_{j \in \text{GPUs}} \mathbf{g}_{ijk},$$

其中 $\mathbf{g}_{ijk}$ 是在工作节点 $k$ 的 GPU $j$ 上拆分的梯度 $i$ 的一部分。这个运算的关键之处在于它是一个 *交换归约*（commutative reduction），也就是说，它把许多向量变成一个向量，而完成向量变换的运算顺序并不重要。这对我们的目的来说是非常好的，因为不需要对何时接收哪个梯度进行细粒度的控制。此外，请注意，这个操作在不同的 $i$ 之间是独立的。

这就允许我们定义下面两个操作：*push*（用于累积梯度）和 *pull*（用于取得聚合梯度）。因为我们有许多层，因此就有很多不同的梯度集合，所以需要用一个键 $i$ 索引梯度。这个键与 Dynamo :cite:`DeCandia.Hastorun.Jampani.ea.2007` 中引入的“键－值存储”之间存在相似性并非巧合。这些定义也满足许多类似的性质，特别是在多个服务器之间分发参数时。

“键－值存储”的 push 与 pull 操作描述如下：

* **push（key，value）**将特定的梯度（值）从工作节点发送到公共存储。在那里，通过将其相加来聚合值。
* **pull（key，value）**从公共存储（例如，组合来自所有工作节点的梯度之后）中取得聚合值。

通过将同步的所有复杂性隐藏在一个简单的 push 和 pull 操作背后，我们可以将统计建模人员（他们希望能够用简单的术语表达优化）和系统工程师（他们需要处理分布式同步中固有的复杂性）的关注点解耦。

## 小结

* 同步需要高度适应特定的网络基础设施和服务器内的连接。这会对同步所需的时间产生重大影响。
* 环同步对于 p3 和 DGX-2 服务器是最佳的，而对于其他服务器则未必。
* 当添加多个参数服务器以增加带宽时，分层同步策略可以工作的很好。

## 练习

1. 你能进一步提高环同步的性能吗？（提示：你可以双向发送消息。）
1. 在计算仍在进行中，可否允许执行异步通信？它将如何影响性能？
1. 如果在长时间运行的计算过程中丢失了一台服务器，该怎么办？如何设计一种容错机制来避免重启计算？

[Discussions](https://discuss.d2l.ai/t/2807)

