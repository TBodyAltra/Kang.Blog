---
date: '2025-01-26T15:39:20+08:00'
draft: false
tags: ["AMDGPU", "GPGPU", "Triton", "Flash-Attention"]
ShowToc: true
title: '在 AMDGPU 上优化 Triton Flash-Attention'
---
最近在 AMDGPU 上优化用 Triton 实现的 Flash-Attention 算子，有一些优化手段值得记录下来。

# 通过调整 Block 发射顺序减少 SIMD 的 IDLE 时间
FA 的 Triton 实现中，将 Q 在 M 方向切分为了不同的 block。在前向过程中，如果 causal = True，那么 Q 只有左下三角的元素参与计载。即参与计算的元素在 M 方向从上到下逐渐增加。在默认的实现中，block 是从上到下按序发射的，即先发射负载小的块，再发射负载大的块。由于负载较大的块难以被分配到 SIMD 上，因此导致了较大的 SIMD IDLE。通过从倒序从下到上发射块，即先发射负载大的块，再发射负载小的块，由于负载小的块可以被更均衡地分配到各 SIMD 上，因此可以有效减少 SIMD IDLE。

| {% asset_img large_idle.jpg large idle %} |  
|:--:|  
| *先发射负载小的块，再发射负载大的块，导致较大的 SIMD IDLE* |  
  
| {% asset_img small_idle.jpg small idle %} |  
|:--:|   
| *先发射负载大的块，再发射负载小的块，可以减少 SIMD IDLE* |  

# 通过实现 chain-dot 减少对 LDS 的访存
在我们的硬件规范下，Q 和 K 矩阵乘的结果 QK 的 Layout 跟 Q 是不同的，因此需要先将 QK 的 Layout 转到跟 Q 一样才可以继续与 V 进行矩阵乘（和 Q 一样作为第一个操作数）。可以通过插入一些寄存器指令对线程之间的数据进行交换，以避免通过写入写出 LDS 来进行 Layout 的转换，这些指令（例如 bpermute, swizzle 等）的开销远小于 LDS 访存。
  
# 通过优化 Layout 增加 Global Memory 访存效率
在我们的硬件规范下，mmac 的结果的 Layout 对于 Global Memory 的访存不太友好。因为在这个 Layout 下，同一个线程所占有的元素在矩阵中的地址不连续，如果直接保存的话，访存效率很低。例如假设结果的数据类型是f16，那么生成的汇编将会是 global_store_short（每条指令仅保存一个元素）。通过在 LDS 中将 Layout 进行转换，使得同一个线程所拥有的元素在矩阵中的地址尽量连续，可以使得汇编中生成 global_store_dwordx4（每条指令保存16个byte，也就是8个元素）。

# 通过调整 Grid 排布增加 L2 Cache Hit

# 重新实现 Torch 的部分 Kernel
Torch 内嵌的操作在我们的硬件上效率比较低，比方说 Transpose 等操作，通过重新实现这些函数的核函数取得了一些性能上的提升。

# 使用 Buffer Load/Store 替换 Global Load/Store
Buffer 可以传递 cache modifier，也可以通过配置 rsrc modifer 来增加 L1 Cache 的命中率（尚未实验）。

# 在 Global Load/Store 指令中使用 SGPR 基地址而不是 VGPR 基地址
因为 SGPR 指令比 VGPR 指令更便宜。

# 对矩阵进行必要的转置以提高访存效率
使得对矩阵的访问尽量是内存地址连续的。

# 外提循环不变量以减少 VGPR 压力
