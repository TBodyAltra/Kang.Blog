---
date: '2025-01-17T11:56:20+08:00'
draft: false
tags: ["AMD", "GPGPU", "Flash-Attention"]
ShowToc: true
title: '用 MUBUF 替换 FA Kernel 中的 GLOBAL 指令引起了概率错误'
---

最近在优化用 triton 写的 flash-attention 的算子的性能，有一个优化是用 MUBUF 指令（buffer_load_dowrdx4，buffer_store_dwordx4 等）替换 GLOBAL 指令（global_store_dwordx4，global_store_dwordx4等），因为通过利用 MUBUF 的 swizzle 的特性可以增加对 L1 Cache 的利用率，同时 MUBUF 指令可以传递一些有用的 cache modifer。

替换完成之后，在 Z 卡上验证精度没有问题，但是在 B 卡上却出现了概率性的精度问题。起初怀疑是在 B 卡上的编译结果有问题，但是直接将 Z 卡上的汇编文件编译到目标为 B 卡的二进制之后，在 B 卡上仍然有精度问题。

仔细对比了 MUBUF 指令替换前后的 LLVM IR，但是看不出来有任何问题。又对比 golden 数据和 triton 的结果，发现错误的数据存在一些规律，但是错误的坐标和数值也存在随机性。于是不再纠结数据是否正确，转而去看是什么原因导致了结果的随机性。有个这个目标后，不断地简化 flash-attention 的实现，直到只剩下几行代码可以稳定地复现随机性。代码简化后，汇编文件也很简单了，直接在汇编文件上修改，同时观察结果。最终发现了出现问题的 pattern:

```LLVM
buffer_store_dwordx4 v[0:3], v32, s[4:7], 0 offen
v_add_u32_e32 v83, s2, v82
v_lshlrev_b32_e32 v0, 1, v33
```

第一行的 buffer_store_dwordx4 将 v[0:3] 的数据写出，第三行 v0 被覆写。虽然根据 ISA 文档，这个 pattern 不存在 data harzard，但是如果在 buffer_store 之后插入一个 nop，精度问题就不存在了。

```LLVM
buffer_store_dwordx4 v[0:3], v32, s[4:7], 0 offen
s_nop 0
v_add_u32_e32 v83, s2, v82
v_lshlrev_b32_e32 v0, 1, v33
```

了解到最近固件的更新会导致非常相似的问题，因此我们怀疑是这个问题同样是由于固件更新导致的。本身不是由 MUBUF 指令引起的，应该是正好撞到了这个 pattern。
