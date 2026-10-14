# 借鉴 ArcReel：LocalMiniDrama 差距分析与落地方案

> 日期：2026-10-14 · 状态：方案稿（待评审）
> 背景：对照开源项目 [ArcReel](https://arc-reel.com/zh/)（AGPL-3.0，AI 短剧/漫剧生成工作台，GitHub 5.2k★）拆解其关键设计，评估 LocalMiniDrama（v1.2.8，MIT）的差距并给出落地优先级。

---

## 一、结论摘要

LocalMiniDrama 已对齐 ArcReel 的大部分核心能力（资产库、一致性、供应商矩阵、工程导出），**真正存在差距的只有三件事**：

| 优先级 | 事项 | 性质 |
|:---:|------|------|
| **P0-1** | 剪映草稿导出 | 功能差距最大，直接提升交付体验 |
| **P0-2** | 任务级断点续传 | 长剧集生成的稳定性刚需 |
| **P1** | 分镜生图自动锚定资产参考图 | 一致性机制的"最后一公里" |
| P2 | 项目级成本追踪 | 可选，契合高性价比诉求 |
| P3 | 多智能体编排 | 观望，单机项目复刻性价比低 |

---

## 二、现状对照表

| 维度 | LocalMiniDrama（已有） | ArcReel | 差距判定 |
|------|------|------|------|
| 资产库 | 角色/场景/道具三库 + 全局素材库 + 「加入本集」 | 三类资产库 + 线索清单 | ✅ 基本对齐 |
| 一致性 | 分镜 `@图片N` 多图参考、首尾帧衔接（`continuity_snapshot`）、ModelArk 资产认证 | 角色 DNA 基准图先行，每张分镜默认锚定 | ⚠️ 缺"自动装配"：参考图主要靠手选 |
| 任务机制 | `async_tasks` 状态表、单步重试（≤3 次）、一键流水线"跳过已有内容"、分镜 JSON 截断自动续写 | lease 调度、断点续传、失败只补失败项 | ⚠️ 缺任务级断点续跑 |
| 版本管理 | 视频历史版本 | 每次重生存档、一键回滚、多版本对比 | ⚠️ 可扩展（分镜图/剧本版本） |
| 导出 | 工程 ZIP（project.json）、分镜表 HTML、解说 SRT | 剪映草稿包直出 + FFmpeg 合成 | ❌ **缺剪映草稿** |
| 编排 | 一键流水线 + 画布工作流（线性 runner） | 主 Agent + Subagent 多智能体 | ⚠️ 单机不必复刻，可借鉴摘要思想 |
| 供应商 | 通义/火山/可灵/Gemini/Agnes/Vidu/NanoBanana/Ollama | 11 家 100+ 模型 | ✅ 基本对齐，本地化更强 |
| 成本 | 无费用统计 | 项目/供应商级费用估算与实际 | ❌ 缺 |

---

## 三、P0-1 剪映草稿导出（最大功能差距）

### 目标
按集导出"剪映草稿包"，用户导入剪映桌面版 5.x / 6+ 后手工精剪、配乐。素材全部现成：分镜视频片段（`video_generations`）、对白 TTS（`audio_local_path`）、旁白（`narration_audio_local_path`）、解说 SRT（已有）。

### 剪映草稿格式要点（社区逆向格式，随版本可能微变）
- 草稿目录 = `draft_content.json` + `draft_meta_info.json` + `resources/`（素材平铺）
- `draft_content.json` 核心结构：
  - `materials.videos / audios / texts`：素材清单（含 `path`、时长）
  - `tracks`：视频轨 / 音频轨 / 字幕轨数组
  - 每轨 `segments`：`material_id` + `target_timerange`（时间线位置）+ `source_timerange`（素材内裁剪区间）
  - `timeline`：总时长、分辨率（9:16 / 16:9 / 1:1 由项目画幅决定）

### 实现路径
1. 新增 `backend-node/src/services/jianyingExportService.js`
   - 入参：`episodeId`
   - 遍历该集 `storyboards`（按 `storyboard_number` 排序）
   - 每个分镜取最新 completed 视频片段 → 依次生成 timeline 段（首尾相接，可读 `duration` 累加）
   - 对白音频 / 旁白音频挂到音频轨；字幕文本（`dialogue`/`narration` + 时间点）生成字幕轨
   - 输出 ZIP：`draft_content.json`、`draft_meta_info.json`、`resources/*`
2. 路由：`GET /api/episodes/:id/export-jianying`（可参考 `videoMerges` 路由的下载形态）
3. 前端：剧集详情页（`DramaDetail.vue`）增加「导出剪映草稿」按钮
4. 验证方式：本地生成草稿包 → 剪映导入 → 检查时间线、视频/音频/字幕三段是否就位

### 风险与对策
- 草稿格式为社区逆向、版本兼容有风险：**先做最小可用版**（视频轨 + 音频轨 + 字幕轨），文档标注"兼容剪映 5.x/6+，以实际导入为准"；后续按用户反馈迭代
- 视频片段时长与 TTS 时长不齐：字幕/音频按片段实际时长做裁剪对齐（`source_timerange` 控制）

---

## 四、P0-2 任务级断点续传

### 现状
- 已有 `async_tasks` 表（`id/type/status/progress/message/resource_id/error/result`）与 `taskService`（`createTask/getTask/updateTaskStatus/cancelTask`）
- 前端 `generationTaskStore`（657 行）+ `useGenerationTaskSync`（151 行）负责轮询
- **缺口**：任务中断（关软件/断网/崩溃）后没有恢复入口；一键流水线靠"生成前检查是否已有"近似跳过已完成项，但无失败项清单与续跑状态机

### 方案
1. **表结构扩展**：`async_tasks` 增加
   - `payload TEXT`：续跑所需的完整参数快照（如 `{dramaId, episodeId, mode}`）
   - `attempts INTEGER DEFAULT 0`：防死循环
   - 复用现有 `error` 字段
2. **启动续跑扫描**：`server.js` 启动时查询 `status IN ('pending','running') AND deleted_at IS NULL` 的任务 → 统一标记 `interrupted` → 日志汇总
3. **恢复接口**：`POST /api/tasks/:id/resume`
   - 语义：读取 `payload` → 按 resource 粒度收集"未完成 / 失败"的分镜 → **只补这些**（复用一键流水线的"跳过已有内容"逻辑，下沉为续跑核心）
   - 前端：进入页面时若检测到 `interrupted` 任务，弹窗「检测到未完成任务，是否续跑」
4. **失败只补失败项**：流水线完成时把失败项写入任务 `result`（失败清单），后续"补全并生成"直接消费该清单

### 边界
- 单步重试（≤3 次）与轮询重试保留不动
- 续跑不重复调用已成功的生成（延续现有"不为已成功内容重复付费"原则）

---

## 五、P1 分镜生图自动锚定资产参考图

### 现状
- 分镜与角色/场景/道具已有绑定（导出代码中已有 `character_indices / scene_index / prop_indices`）
- 分镜生图参考图部分手选（`@图片N`）；Roadmap 有"参考图自由选择"

### 方案（把 ArcReel "角色 DNA" 用现有数据结构落地）
1. **默认装配**：生成分镜图时，自动把该分镜绑定的**角色图 + 场景图 + 道具图**作为参考图注入提示词（`@图片1 @图片2 …`），无需用户手选；用户仍可手动增删
2. **按模型能力裁剪**：不同模型多图参考上限不同（Seedance 2.0 支持多图，部分模型仅 1–2 张）——在 `frontweb/src/utils/modelSelection.js` 增加"参考图数量上限"映射，按当前模型裁剪优先级（角色 > 场景 > 道具）
3. **资产变更联动**：角色/场景/道具图被重新生成后，标记依赖该资产的分镜为"待重生成"（新增 `storyboards.needs_regenerate` 或复用现有状态字段），画布上提供"批量重跑受影响分镜"入口
4. 注意：尾帧衔接（`storyboards_tail_link`）与自动装配并存时，参考图顺序要稳定（首帧参考 → 资产参考 → 尾帧参考），避免提示词拼接错位

---

## 六、P2 项目级成本追踪（可选）

- `image_generations` / `video_generations` 已有 `provider / model / size / quality` 字段
- 新增 `cost_estimate REAL` 列：按"模型单价表 × 用量"估算（单价表放 `config.yaml` 或 `constants/`，可按供应商维护）
- 前端：项目详情页加"费用汇总"卡片（按集/按供应商/按类型），生成前给出估算
- 价值：契合本项目"接入多家供应商、按需选模型"的定位，帮用户省 API 钱

---

## 七、P3 多智能体编排（观望）

- 单机本地项目复刻"主 Agent + Subagent"架构性价比低（需持续调用大模型做编排，成本与复杂度高）
- **可借鉴的思想**：只在"小说/长文导入"环节（`novelImportService`、`dramaImportService`）借鉴"大上下文摘要化"——先对原文做章节/场景结构化摘要，再进入分集与分镜，避免长上下文塞爆提示词
- 结论：暂不立项，保持线性工作流 + 画布 runner 架构

---

## 八、工作量粗估（相对人日，供排期参考）

| 事项 | 后端 | 前端 | 验证 |
|------|:---:|:---:|:---:|
| 剪映草稿导出 | 2–3 | 0.5 | 剪映实机导入 |
| 断点续传 | 1.5–2 | 1 | 杀进程重启续跑 |
| 自动锚定资产 | 0.5–1 | 1 | 多模型回归 |
| 成本追踪 | 1 | 0.5 | 抽样核算 |

---

## 九、参考来源

- ArcReel 官网：https://arc-reel.com/zh/
- ArcReel 第三方评测（Aitoolnet）：https://www.aitoolnet.com/arcreel
- 腾讯云实践（ArcReel + 免费 API 流水线）：https://cloud.tencent.com/developer/article/2684247
- FollowAgents 评级（52/100，口径为 agent 框架能力）：https://followagents.com/en/agents/arcreel
