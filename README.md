# SeedVR2 本地部署 · 种子文档

> RTX 4070 Super（12GB VRAM）本地部署 **SeedVR2** 视频修复 / 超分模型的完整说明。
> 文档遵循 knowledge-base [「单项目规范」](https://github.com/Simiely/knowledge-base/blob/main/模板库/单项目规范/README.md) 四件套：README / AGENTS / DEVELOPMENT / CHANGELOG。

## 这是什么

**SeedVR2**（ByteDance Seed，Apache-2.0）是一个**盲视频修复 / 超分**模型，有 3B / 7B 两档。它在 12GB 消费卡上的推荐档位是 **3B FP8 量化**。

它能做的：
- ✅ 把 720p / 有噪 / 轻度模糊的视频修到 **约 1080p**，时序一致、一步式推理（比多步扩散快）
- ✅ 去噪、去模糊、修锯齿、放大分辨率

它**不能**做的：
- ❌ **不接受提示词**——节点里没有 prompt 输入框，无法按文字加风格 / 加效果 / 让画面动起来
- （提示词类视频效果属于 Wan / LTX / MiniMax H3 等生成模型，不在本仓库范围）

## 能力边界（先看清再装）

| 你的目标 | 是否适合本仓库 |
|---|---|
| 把已有视频变清晰 / 修复瑕疵 | ✅ 这是它的定位 |
| 输入到更接近 2K | ⚠️ 12GB 现实产出约 1080p 档 |
| 按提示词加风格 / 加对白音效 | ❌ 换生成模型（Wan / LTX / MiniMax H3） |
| 商用 | ❌ 参考各模型许可证 |

## 安装

- 主程序：**ComfyUI Windows portable（NVIDIA）­· v0.33.1**
- 节点：`ComfyUI-SeedVR2_VideoUpscaler`（版本见 AGENTS.md）
- 权重（放 `ComfyUI/models/SEEDVR2/`）：
  - `seedvr2_ema_3b_fp8_e4m3fn.safetensors`（约 3.16GB，必备）
  - `ema_vae_fp16.safetensors`（约 478MB，必备）
  - 3B/7B 其余档位可选（7B 需 ≥24GB）
- 依赖：`diffusers 0.40` / `peft 0.20` / `gguf` / `matplotlib` / `opencv` / `omegaconf` / `rotary-embedding-torch`

> 节点首次运行会自动把必备权重下载到 `ComfyUI/models/SEEDVR2/`。网络受限时需走代理，见 AGENTS.md。

## 快速开始

1. 双击启动脚本 `run_nvidia_gpu.bat`
2. 浏览器打开 `http://127.0.0.1:8188`
3. 把 `workflows/seedvr2_upscale_1080p.json`（或快档 `_res1024_batch9`）**直接拖进画布**
4. 在 `LoadVideo` 节点选视频 → 点 **Queue**

## 性能 / 提速

针对 12GB 卡的核心结论：
- **用 3B FP8 + VAE 分块（tiling）+ DiT `blocks_to_swap`**；这是基准数据验证的"效率路径"（小 batch + tiling + block-swap）。
- **别用大 batch**：大 batch 反而慢 / 爆显存，建议 `batch_size=9` 或更小。
- 详细参数取舍见 [DEVELOPMENT.md](DEVELOPMENT.md)。

## 文档索引

| 文件 | 给谁看 | 内容 |
|---|---|---|
| `AGENTS.md` | AI / 未来的你 | 精确版本、关键坑、约定、常用命令、文档基线 |
| `DEVELOPMENT.md` | 开发者 | 架构说明 + 关键问题记录（一坑一篇） |
| `CHANGELOG.md` | 所有人 | 版本变更 |
| `workflows/` | 使用者 | 可直接 Load 的 ComfyUI 工作流（.json） |