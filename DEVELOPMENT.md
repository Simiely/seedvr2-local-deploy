# DEVELOPMENT.md · 架构与问题记录

> 结构固定三块：项目概览 → 架构说明 → 关键问题与方案（一坑一篇，格式与 knowledge-base 完全一致，提炼零转换）。

## 一、项目概览

- **目标**：在 RTX 4070 Super（12GB）上本地跑通 SeedVR2-3B FP8 的视频修复 / 超分。
- **结论**：可稳定产出（非 MiniMax H3 那种"能装不能用"）。现实产出落在大约 1080p，时长时延约数十秒/短片。
- **技术栈**：ComfyUI portable + SeedVR2 节点 + 3B FP8 权重；VAE 分块 + DiT block-swap 撑起 12GB。

## 二、架构说明

- **数据流**：
  `LoadVideo → GetVideoComponents → SeedVR2VideoUpscaler(DiT + VAE) → CreateVideo → SaveVideo`
- **12GB 关键配置组合**：
  - 3B FP8 权重（省显存的核心）
  - VAE tiled 编码，tile=1024，overlap 128（快档可降到 64）
  - DiT `blocks_to_swap=24`（去噪时把部分层换到 CPU，12GB 才能撑）
  - 小 `batch_size=9`（配合上述才有稳定速度）
- **两版工作流**：
  - `seedvr2_upscale_1080p.json`：res 1080、batch 33、overlap 128——兼容稳妥版
  - `seedvr2_fast_res1024_batch9.json`：res 1024、batch 9、overlap 64——快档版

## 三、关键问题与方案（一坑一篇）

### 问题：VAE 编码极慢（单 batch 约 8 分钟）

**TL;DR**：VAE 被 `offload_device=cpu` 丢到了 CPU 上算，12GB 卡叠加 1024×1024 分块后灾难性降速。

- 现象：日志 `Encoding batch N/29` 每次间隔 5~8 分钟，UI 进度条停在 0%。
- 根因：`SeedVR2LoadVAEModel` 的 offload device 设成了 CPU。
- 解决：VAE 的 `offload_device` 改 `cuda:0`，让编码上 GPU。
- 预防：慢时先看任务管理器——GPU 占用低且 CPU 占用高 = offload 设错了。

### 问题：大 batch_size 反而更慢 / 爆显存

**TL;DR**：SeedVR2 的消耗主力是 **VAE 解码分块**，不是模型大小或分辨率；大 batch 逼近显存天花板，触发 OOM / 分页拖慢。**小 batch 才高效。**

- 现象：845 帧、batch 33 时全程预计达数小时量级。
- 根因：参考 `ai-upscaler-benchmark`（2026-06）实测：1080p 下 **batch=9 已约 0.11 fps**，batch 更大/更高分辨率直接 OOM；它给出的"效率路径"= 小 batch + VAE tiling + block-swap。[benchmark](https://github.com/ismael-joffroy-chandoutis/ai-upscaler-benchmark/blob/main/docs/video-upscalers.md)
- 解决：`batch_size` 降到 9（或 5）；保留 VAE tiling + `blocks_to_swap`。
- 预防：长视频优先小 batch + tiling，别贪大 batch。

### 问题：`resolution` 参数填 `"fixed"` 不生效

**TL;DR**：该字段要整数；填字符串会走默认值、控件显示异常。

- 现象：节点分辨率控件为字符串值，输出分辨率不符合预期。
- 解决：显式填整数 `1080`（或快档 `1024`）。
- 预防：改完工作流用文件拖进画布重载，看节点实际显示的值确认。

### 问题：每次跑都像第一次一样慢

**TL;DR**：`cache_model=false` 时每次任务结束就卸载模型，下一次重新加载 + 冷启动。

- 现象：换第二个视频又开始漫长的加载阶段。
- 解决：Loader 节点 `cache_model` 开 `true`，模型常驻，后续任务直接复用。
- 注意：会持续占用显存（3B FP8 约 3GB）。
- 预防：连续修多段视频时开；单次任务可不开。

### 问题：git / curl 不走系统代理，GitHub 直连被重置

**TL;DR**：本机代理是 `127.0.0.1:7890`，但 `git`/`curl` 默认不读系统代理；git 协议连 GitHub 偶尔直接失败。

- 现象：`git clone` 卡住 / 被 reset；权重下载失败。
- 解决：`git`/`curl` 显式 `-x http://127.0.0.1:7890`；克隆节点改用 **zip 包**而非 git 协议。
- 预防：下载节点 / 权重前先确认代理链路能通。

## 四、性能基准参考

- **外部基准**：`ai-upscaler-benchmark`（2026-06，测于 RTX 5090）——SeedVR2 1080p batch 9 ≈ **0.11 fps**；结论是"VAE 解码分块主导消耗"，效率路径 = 小 batch + VAE tiling + blockswap。[benchmark](https://github.com/ismael-joffroy-chandoutis/ai-upscaler-benchmark/blob/main/docs/video-upscalers.md)
- **4070S 现实预期**：845 帧、1080p 档全程约 **30~60 分钟**量级是合理水平（非故障）；想更快 → `resolution=1024` + `batch=9`（快档版即此组合）。
- **判断是否卡死**：看日志 `Encoding batch N/29` 的 N 是否递增、任务管理器 GPU 利用率是否高；批次号递增且 GPU 忙 = 在跑，耐心等。