# v2-port 移植备忘：跳过项及上游对应实现

> 对应《Ombre Brain 新版移植工单（v2-port）》，2026-07-04 执行。
> 底座：upstream/main @ 740d8c6（v2.4.13）。旧 fork 完整存档于 `legacy-backup` 分支。
> 本分支四个移植 commit：§1.1 ae06dad · §1.2 adc37da · §1.3 de0e7c4 · §1.4 1db9abc。

## 一、已移植项（4 个 commit）

| 工单条目 | commit | 内容 |
|---|---|---|
| §1.1 三段提示词 | ae06dad | DEHYDRATE/DIGEST/MERGE 逐字换回旧 fork 自然语气版；保留上游结构项（视角铁律指针、DIGEST tags 两步生成）；`_PROMPT_VERSION` 2→3 使旧缓存失效 |
| §1.2 valence 衰减 | adc37da | `emotion_weight = base + arousal×arousal_boost + max(0, valence−0.5)×valence_boost`；valence 读 metadata、缺省 0.5、clamp 0~1；新 config 键 `decay.emotion_weights.valence_boost`（代码默认 0.6） |
| §1.3 配置搬家 | de0e7c4 | 根目录 `config.yaml` 成品（见下方对照表；`add -f` 跟踪，沿旧 fork 惯例；密钥不入库） |
| §1.4 thinking 抑制 | 1db9abc | `_chat_once` 的 openai_compat 分支对 Gemini 端点注入 `extra_body.extra_body.google.thinking_config.thinking_budget`（复用 `dehydration.thinking_budget`，默认 0；None 不发送） |

## 二、跳过项（新版已内置同等能力）

| 工单条目 | 上游对应实现 | 位置 |
|---|---|---|
| §1.4 503/429 指数退避重试 ×3 | `_chat` 统一重试：429/500/502/503/504 + timeout/connect/ratelimit/unavailable，退避 0.8×2^n，覆盖面比旧 `_create_with_retry` 更广 | `src/dehydrator.py` `_chat` / `_RETRY_*` |
| §1.4 日记整理 max_tokens 8192 | `_DIGEST_MAX_TOKENS = 8192`（另有 `_ANALYZE_MAX_TOKENS = 4096`） | `src/dehydrator.py` |
| §1.5 API-only 显式报错 | `_require_api()` 统一抛 RuntimeError；analyze/merge/digest 失败也转 RuntimeError → hold/grow 显式报错。验收：无 key 时四链路全部 RuntimeError ✓ | `src/dehydrator.py` |
| P1 B-a resolved 不立即归档 | `update()` 已移除 resolved→archive 分支，注释与旧 fork 一致 | `src/bucket_manager.py:709` |
| P1 B-b resolved 关键词可达 | 阈值用降权前原始分，×0.3 仅作用排序；另增强：literal 命中直接放行 | `src/bucket_manager.py:1042` |
| P1 B-c activation_count 从 0 起 | `"activation_count": 0` | `src/bucket_manager.py:410` |
| P1 B-d feel 空 domain | `bucket_type == "feel"` 允许空 domain | `src/bucket_manager.py:384` |
| P1 B-e 时间分 exp(−0.02×days) | `TIME_DECAY_LAMBDA = 0.02` | `src/bucket_scoring.py:43` |
| §1.2 附带：activation_count float（B-03） | `max(1.0, float(...))` | `src/decay_engine.py` calculate_score |
| §1.2 附带：auto-resolve 同轮 meta 刷新（B-08） | `meta["resolved"] = True`（注释同旧 fork） | `src/decay_engine.py` run_decay_cycle |
| （旧 fork B-09）hold 用户 valence/arousal 优先 | `final_valence = valence if 0<=valence<=1 else 打标值` | `src/tools/hold/core.py:53` |
| （旧 fork B-06/B-07）w_time 1.5 / content_weight 1.0 | 代码默认已改；config 成品仍显式写死防漂移 | `src/bucket_manager.py:196,198` |
| P2 check_icloud_conflicts.py | 上游已收录，内容与旧 fork 逐字一致（仅行尾 CRLF/LF 差异）。注意：脚本移居 `tools/` 后其「同目录 config.yaml」定位失效，需用 `--buckets-dir` 或 `OMBRE_BUCKETS_DIR` | `tools/check_icloud_conflicts.py` |

## 三、放弃项（用户拍板 / 工单 P3）

- **Webhook 推送**（OMBRE_HOOK_URL/OMBRE_HOOK_SKIP、_fire_webhook 及各挂点）：放弃，不移植（2026-07-04 拍板，Render 从未配置）。
- P3 全部：dashboard 魔改/auth 全套/重定向（新版 `frontend/` + `src/web/` 已重写，含密码向导——本地已实测 setup/login/401 保护均正常）、embedding 预筛层（新版七维评分原生含 semantic+bm25 双通道）、import 修补（新版导入器重写）、文档/compose 注释/旧测试套件/reclassify 等小补丁（legacy-backup 永久可查）。
- P3「确认再弃」核对结果：`OMBRE_PORT` 新版原生（默认 18001，Docker 固定 8000）；`OMBRE_DASHBOARD_PASSWORD` 新版原生（env 覆盖文件密码）；`OMBRE_*_MODEL` → 新名 `OMBRE_COMPRESS_MODEL` / `OMBRE_EMBED_MODEL`。均可弃。

## 四、行为差异说明（上游新设计，非工单偏离）

1. **dehydrate 瞬时失败降级**：无 key → 仍抛 RuntimeError（符合 §1.5）；但「有 key、重试 3 次后仍失败」时返回原文截断片段并带「原文截断·脱水暂不可用」标记（不写缓存，恢复后自动重压）。仅影响 breath/dream 展示链路，hold/grow 仍显式报错，无关键词提取、不产生错误记忆。
2. **embedding 成为写入硬依赖**：`create()/update(content=…)` 前置校验 embedding 可用，否则拒绝写入（不再允许「文件存在但向量缺失」）。⚠️ 部署必须配 `OMBRE_EMBED_API_KEY`，否则 hold/grow 直接报错。
3. **视角铁律**：脱水/合并 prompt 后自动附加，把原文人称还原为 `human` 配置的名字（成品已设 `human: "UU"`，Dashboard 可改）。
4. **config.yaml 会被运行时规范化重写**（排序、去注释）：属上游「启动时环境变量落盘」机制，值不变，仅格式。

## 五、部署注意（Render 环境变量改名）

| 旧 fork | 新版 | 说明 |
|---|---|---|
| `OMBRE_API_KEY`（脱水+向量共用） | `OMBRE_COMPRESS_API_KEY` + `OMBRE_EMBED_API_KEY` | **不再互相复用**，两个都必须设（可以是同一个 Gemini key） |
| `OMBRE_DEHYDRATION_MODEL` / `OMBRE_MODEL` | `OMBRE_COMPRESS_MODEL` | 同理 base_url → `OMBRE_COMPRESS_BASE_URL` |
| `OMBRE_EMBEDDING_MODEL` / `_BASE_URL` | `OMBRE_EMBED_MODEL` / `OMBRE_EMBED_BASE_URL` | |
| `OMBRE_BUCKETS_DIR` | 兼容保留，推荐新名 `OMBRE_VAULT_DIR` | |
| `OMBRE_TRANSPORT` / `OMBRE_PORT` / `OMBRE_DASHBOARD_PASSWORD` | 同名保留 | |

## 六、config.yaml 键对照表（§1.3）

| 工单键 | 用户值 | 新版键 | 上游默认 | 处置 |
|---|---|---|---|---|
| decay.lambda | 0.03 | 同名 | 0.05 | config 写入 |
| decay.threshold | 0.25 | 同名 | 0.3 | config 写入 |
| decay.emotion_weights.arousal_boost | 1.2 | 同名 | 0.8 | config 写入 |
| decay.emotion_weights.valence_boost | 0.6 | 同名（§1.2 新增） | —（移植后代码默认 0.6） | config 写入 |
| scoring_weights.time_proximity | 1.5 | 同名 | **1.5（已同值）** | 显式写死防漂移 |
| scoring_weights.content_weight | 1.0 | 同名 | **1.0（已同值，B-07 已吸收）** | 显式写死防漂移 |
| dehydration.model | gemini-2.5-flash | 同名 | 示例 deepseek-chat / 代码 gemini-2.0-flash | config 写入 |
| dehydration.base_url | …googleapis…/v1beta/openai | 同名 | api.deepseek.com/v1 | config 写入 |
| transport | streamable-http | 同名 | stdio | config 写入（OMBRE_TRANSPORT 亦可） |
| —（新增） | 0 | dehydration.thinking_budget | 0 | 显式写入（§1.4 文档化） |
| —（新增） | UU | human | "用户" | config 写入（视角铁律称呼，2026-07-04 拍板） |
| —（新版原生段） | 上游默认 | mcp_require_auth / hooks / surfacing / limits / bucket_type_defaults | — | 保持默认（sampling.enabled=false 与旧版浮现行为一致） |
