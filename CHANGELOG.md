# CHANGELOG

## 2026-11-18 · 文档维护：接入 repo-kit 机器层（link-only）

- `schema/taxonomy.json` + `schema/kit.lock.json`（kit `2.9.0` / 规范 `2.1.4`）+ `tools/repo-kit/check_repo.py`
- `.github/workflows/repo-check.yml`：md / `schema/**` / `tools/**` 变更时跑链接与锚点检查 + 锁哈希校验
- 只改文档基建，不动产品代码；验收 `REPO CHECK: PASS`（ERROR=0、WARN=0）、`KIT LOCK: PASS`
- 文档基线补上真实 commit 哈希（原「commit 见本仓库首次提交」无法用于断点续传）

## [0.1.0] 2026-08-26

- 首次发布：SeedVR2-3B FP8 在 RTX 4070 Super（12GB）的本地部署方案。
- 交付内容：
  - 四件套文档：README / AGENTS / DEVELOPMENT / CHANGELOG
  - 两版 ComfyUI 工作流：`seedvr2_upscale_1080p.json`、`seedvr2_fast_res1024_batch9.json`
  - 关键参数踩坑记录（VAE offload、batch 大小、resolution、代理、cache_model，见 DEVELOPMENT.md）
- 范围说明：本仓库仅覆盖 SeedVR2 视频修复/超分；提示词类视频生成（Wan / LTX / MiniMax H3）不在范围内。

### 已知限制

- 12GB 卡现实产出约 1080p；更高/更长需 24GB+ 或换模型。
- 不接收提示词，无法按文字加风格/效果。