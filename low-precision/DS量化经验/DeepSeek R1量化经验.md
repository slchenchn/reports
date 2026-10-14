# DeepSeek R1 量化实战经验分享

最近基于DeepSeek-R1-671B入门模型量化，这里记录分享一些经验。

相比其他主流大模型（如Qwen系列），DeepSeek R1具有一些独特的特性：
- 模型规模巨大，671B的参数量，8x80G的显存也放不下，而且是非常稀疏的MOE结构，对一些需要校验集的方法，例如GPTQ，并不友好
- R1本身是用低精度训练的，对量化更加鲁棒，而且token/param比例为14.8T/671B = 22.05，接近chinchilla scaling law的compute-optimal，理论上量化损失更小


## 硬件配置
- GPU: 8 x A800
- 内存：1T

## W4A16
目前W4A16量化已经是比较成熟的技术路线，主要采用[AWQ](https://github.com/casper-hansen/AutoAWQ)方法，有开源代码和社区支持，也有[量化好的模型供](https://huggingface.co/cognitivecomputations/DeepSeek-R1-AWQ)下载。我这个配置下，稍微限制校验集大小，AutoAWQ也能直接量化。

我这边测试的结果是：量化结果基本是无损的，精度下降在1%以内。

部署AWQ模型时，建议使用vllm，dtype设为bf16，fp16会导致数值溢出，详见[我另一篇文章](https://zhuanlan.zhihu.com/p/1898058518684218923)。

值得一提的是，W4A16在低负载时有不错的解码效率，但在高负载下，其效率快速降低，甚至远不如bf16的版本。
一方面是因为高负载下memory-bound逐步转变为compute-bound，另一方面，目前主流的R1推理优化还是聚焦于bf16。


## W8A8
如前所述，R1其实是比较好量化的，因此美团的[RTN量化](https://huggingface.co/meituan/DeepSeek-R1-Block-INT8)都能取得无损的效果。


## W4A8
W4A8下，RTN就有点不太够了，例如GPQA和LiveCodeBench，精度会有比较明显的下降。
目前也没有开源的W4A8权重，需要自行量化。
下面介绍两个使用过的量化框架

### llm-compressor

[llm-compressor](https://github.com/vllm-project/llm-compressor)是vllm官方的量化框架，支持SmoothQuant和GPTQ。
但这个框架没有做动态offload，模型权重所占空间、hessian矩阵所占空间都预先计算且静态分配，导致面对671B的模型，会几乎将全部层都放在CPU中，导致几乎所有操作都在CPU上进行，再加上R1这种超稀疏的MOE，量化的速度实际非常慢。

并且这个框架抽象层次太多（session-lifecycle-stage），二来这个框架还用了torch.compile，导致torch、transformers、llm-compressor几种机制搅合在一起，debug困难，因此并不推荐用这个框架来量化R1。

### llmc
[llmc](https://github.com/ModelTC/llmc)的思路就比较适合量化超大模型，他只会将当前需要量化的层搬到gpu0上，量化完了中间结果删除，需要保留的权重再搬回去。

但实测在我的设备上，671B的模型依然难以量化。我也进一步做了以下修改：


#### dispatch到其他显卡
llmc中，将模型放在内存中，只将当前需要量化的层搬到gpu0上，量化完了再搬回gpu0。因此实际可用存储为内存+gpu0的显存，并没有用到其他GPU的显存。
因此我考虑将部分参数offload到gpu1-gpu7上，这样可以额外提供560G的空间，对于不需要校准集的方法，已经足够。

#### fp8
即使将部分参数dispatch到其他显卡，但总存储空间仍是捉襟见肘，难以支撑需要校准集的量化方法。
因此可以采用fp8格式来存储模型权重，从而将这部分显存需求减半。

此外，还有一些小修改，具体可以参考我fork的[llmc](https://github.com/slchenchn/llmc)。
 

经过上述修改后，已经可以做一个粗糙的量化，实测只做offline旋转，也可以实现无损W4A8量化。

此外，还有一些小的点，例如因为MLA比较复杂，实际kv_b_proj和q_b_proj并没有做旋转，这两个可能做 smoothquant 更好。

## 总结


