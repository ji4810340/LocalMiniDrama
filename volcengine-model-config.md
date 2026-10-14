# 火山引擎方舟模型配置指南（LocalMiniDrama）

> 所有模型均走火山引擎方舟（Ark），区域 cn-beijing
> 在应用「AI 配置」页面逐条添加，每条对应一个服务类型

## 公共信息

| 项 | 值 |
|----|----|
| Base URL（文本/图片） | `https://ark.cn-beijing.volces.com/api/v3` |
| Base URL（视频） | `https://ark.cn-beijing.volces.com/api/v3` |
| API Key | 在方舟控制台 → API Key 管理创建 |
| 视频 endpoint | `/contents/generations/tasks` |
| 视频 query_endpoint | `/contents/generations/tasks/{taskId}` |
| 图片 endpoint | `/images/generations` |
| 文本 endpoint | `/chat/completions` |

---

## 方案 A：成本最低

### 1. 文本（剧本/对白生成）

| 字段 | 值 |
|------|----|
| service_type | `text` |
| provider | `openai`（方舟兼容 OpenAI 协议） |
| name | `豆包Seed-2.0-mini（低成本）` |
| base_url | `https://ark.cn-beijing.volces.com/api/v3` |
| model | `doubao-seed-2-0-mini-260428` |
| 单价 | 输入 0.2 元/M，输出 2.0 元/M（≤32k） |

### 2. 图片（角色/场景图）

| 字段 | 值 |
|------|----|
| service_type | `image` |
| provider | `volces` |
| name | `Seedream-5.0-Flash（低价出图）` |
| base_url | `https://ark.cn-beijing.volces.com/api/v3` |
| model | `doubao-seedream-5-0-flash` |
| 单价 | 0.12 元/张 |

### 3. 视频（核心成本项）

| 字段 | 值 |
|------|----|
| service_type | `video` |
| provider | `volces` |
| name | `Seedance-2.0-mini（低成本视频）` |
| base_url | `https://ark.cn-beijing.volces.com/api/v3` |
| model | `doubao-seedance-2-0-mini-260615` |
| 单价 | 480p 5秒 ≈ 1.16 元/条（限时4折）；约 0.23 元/秒 |

> 备选更便宜：`doubao-seedance-1-0-pro-250528`（1.0-pro-fast），480p 5秒 ≈ 0.20 元/条，约 0.04 元/秒，但画质较旧。

---

## 方案 B：效果最好

### 1. 文本（剧本/对白生成）

| 字段 | 值 |
|------|----|
| service_type | `text` |
| provider | `openai` |
| name | `豆包Seed-2.1-Pro（旗舰）` |
| base_url | `https://ark.cn-beijing.volces.com/api/v3` |
| model | `doubao-seed-2-1-pro` |
| 单价 | 输入 6.0 元/M，输出 30.0 元/M，1M 上下文 |

### 2. 图片（角色/场景图）

| 字段 | 值 |
|------|----|
| service_type | `image` |
| provider | `volces` |
| name | `Seedream-5.0-Pro（最高画质）` |
| base_url | `https://ark.cn-beijing.volces.com/api/v3` |
| model | `doubao-seedream-5-0-pro` |
| 单价 | 0.30~0.60 元/张（支持图层拆分） |

### 3. 视频（核心成本项）

| 字段 | 值 |
|------|----|
| service_type | `video` |
| provider | `volces` |
| name | `Seedance-2.5（旗舰视频）` |
| base_url | `https://ark.cn-beijing.volces.com/api/v3` |
| model | `doubao-seedance-2-5-260628` |
| 单价 | 720p 5秒 ≈ 7.56 元/条；1080p 5秒 ≈ 18.71 元/条 |

---

## 成本对比（每集短剧估算）

假设：剧本输入 50k + 输出 20k token；图片 15 张；视频 10 段 × 5 秒。

| 项目 | 方案 A | 方案 B |
|------|--------|--------|
| 文本 LLM | ≈ 0.05 元 | ≈ 0.90 元 |
| 图片 15 张 | 1.80 元 | 6.75 元 |
| 视频 50 秒 | ≈ 11.5 元（480p） | ≈ 75.6 元（720p） |
| **合计/集** | **≈ 13.4 元** | **≈ 83.3 元** |

---

## 推荐日常用法

| 场景 | 选哪个 |
|------|--------|
| 跑草稿、批量出分镜 | 方案 A（视频 480p） |
| 正式出片、关键镜头 | 方案 B（视频 720p/1080p） |
| 复杂剧情大纲/反转逻辑 | 方案 B 文本（开 thinking） |
| 日常对白润色 | 方案 A 文本（够用） |

## 代码变更

已在 `backend-node/src/services/videoClient.js` 补充：
- `VOLC_MODEL_ALIASES` 新增 seedance-2.0 / 2.0-mini / 2.5 的别名映射
- `normalizeVolcengineDuration` 新增 seedance 2.5 的 4–30 秒时长支持
