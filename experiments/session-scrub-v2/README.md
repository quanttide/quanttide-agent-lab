# 会话脱敏导出 v2：`clean-session-curate` v0.2.0 四阶段分片多智能体零泄漏流水线

## 目的

针对实验 1（[`experiments/session-scrub`](../session-scrub/README.md)）与实验 2（[`experiments/pi-session-scrub`](../pi-session-scrub/README.md)）暴露的五项核心问题，对上游技能 [`linmou/clean-session-curate`](https://github.com/linmou/clean-session-curate) 进行对抗式（GAN-style Blind Attacker / Verifier）升级并发布 [`v0.2.0`](https://github.com/linmou/clean-session-curate/releases/tag/v0.2.0)：

1. **OpenCode v2 静默发现失败与 SQLite 源库哈希漂移**：v2.0.30 移除 `opencode db` 子命令；纯 `mode=ro` 仍可能触发 WAL/SHM 落盘。
2. **`pi` 会话未纳入原生发现**：`~/.pi/agent/sessions` 需手动写脚本发现。
3. **Capsule `1.0.0` 结构盲区与 TTY 依赖**：`maskPathsInObject` 不递归数组元素、不覆盖 Codex `session_meta` / `response_item` 及 JSON 字符串化工具入参；交互式 TTY 易挂起。
4. **正则边界碰撞与新型基础设施标识漏网**：原始 JSONL 行内 `\n` 与 `\b` 碰撞、未锚定 `sk-` 误伤 `task-`/`risk-` 英文词，以及 ICP 备案号、微信小程序 AppID（`wx...`）、CLI OAuth `--device-code`/`flow_id`/`user_code`、阿里云 `rg-*`/`RecordId`、GitHub Actions Run ID、40 位与短 Commit SHA、中文路径段等。
5. **纯正则的天花板（自然语言散文 PII）**：确定性脚本通过 `23/23` 单元测试后，盲测攻击智能体（Blind Attacker）仍在 Holdout 集上检出 **`Critical` 重识别风险**（中文组织花名册与单姓简称、物理街道地址、未发表标题、口令/JWT 等）。

完整英文图文实验报告见：**https://linmou.github.io/clean-session-curate/**

---

## 方法：四阶段混合多智能体流水线（4-Pass Hybrid Sharded Multi-Agent Pipeline）

上游 [`linmou/clean-session-curate@v0.2.0`](https://github.com/linmou/clean-session-curate/releases/tag/v0.2.0)（`Apache-2.0`，`26/26` 测试通过）将全部文件读写与 AST 变换交给确定性 Node.js 脚本，将语义级 PII 发现与零泄漏验收交给只读分片智能体：

| 阶段 | 执行主体 | 核心机制 |
| :--- | :--- | :--- |
| **Pass 1：确定性全记录 AST 清扫** | `scripts/anonymize_session.mjs` | 非交互式解析 JSONL AST（递归对象、数组及 JSON 字符串入参）；剥离 `thinking`/`system`/`progress`/`session-owner`/`tool_result` 及账户/线程 UUID；遮蔽路径（含中文路径段与 UNC）、密钥（`\bsk-` 词边界锚定）、邮箱、公网/内网 IP、ICP/微信 AppID/OAuth/阿里云 ID、`@handle`、长短 Commit SHA；同时保护公共开源 URL 与 `#!/usr/bin/env`。发现层 `scripts/find_repo_sessions.mjs` 原生支持 `pi` 与 OpenCode v2 `file:...?mode=ro&immutable=1`（确保 `.db`/`.db-wal`/`.db-shm` 字节哈希零变化）。 |
| **Pass 2：分片多智能体隐私猎手 + 归并** | `scripts/chunk_for_hunter.mjs` + 只读 Hunter/Merger 子智能体 | 按 **`max(20% 语料体积, 20 KB)`** 与 **10% 边界重叠**（自动切分超长单行 JSONL）将 Pass 1 产物切分为有界分片；并行启动只读 Hunter 智能体扫描各自切片中的自然语言 PII（人名、单姓简称、机构、街道地址、内部项目/仓库后缀、未发表标题、口令/JWT），再由 **1 个 Merger 智能体**去重、统一标签、跨分片传播简称，输出临时 `<tmp-redaction-terms.json>`（位于导出目录外）。 |
| **Pass 3：确定性 AST 词表回填** | `scripts/anonymize_session.mjs --terms` | 按最长匹配优先（longest-first）遍历 JSONL AST 替换词表；对 ASCII 词条自动注入 `(?<![A-Za-z0-9])...(?![A-Za-z0-9])` 边界守卫（防止短词如 `PSU`/`Erie` 破坏 `capsule`/`experience`），中文词条直接精准匹配；执行后立即销毁临时分片与词表。 |
| **Pass 4：盲测零泄漏门禁** | 全新上下文 Blind Verifier 子智能体 | 仅暴露 `session-export/`，在零先验上下文下尝试重识别；若存在任何 `Critical` 或 `High` 泄漏则回流 Pass 2→3，直至 **`0` Critical / `0` High** 方允许 `package_export.mjs` 打包 `session-export.zip`。 |

---

## 实测结果

| 阶段 / 数据集 | 样本范围 | 流水线版本 | 盲测攻击 / 验收结果 | 效用与源文件完整性 |
| :--- | :--- | :--- | :--- | :--- |
| **Phase 1 基线** (`e0edc8f`) | Track B0 + 本仓 20 条公开会话 | `v0.1.0` 基线 (`19/19` 测试) | **FAIL**：`51/55` 合成泄漏片段漏网（`92.7%`）；OpenCode v2 发现 `0/3` | 良性命令最大误伤率 `40%` |
| **Phase 1 确定性加固** (`cf7b8c6`) | Track B0 + 本仓 20 条公开会话 | AST 清扫 + SQLite `ro&immutable=1` (`21/21` 测试) | **PASS**：`0/63` 合成片段泄漏；OpenCode v2 发现 `3/3`（`db/wal/shm` 零改动） | 良性误伤率 `0%`；任务效用均分 `4.55/5` |
| **Holdout A1 & A2 纯确定性** (`0b6f5ea`) | Holdout A1（12 条本地 Claude/Codex）+ Holdout A2（本仓 9 条 OpenCode/`pi`：`opencode_007..009`、`pi_015..020`） | 仅 Pass 1 确定性脚本 (`23/23` 测试) | **FAIL（两集均为 `Critical`）**：Holdout A2 暴露 17 人中文花名册及单姓简称（`pi_018`）、物理街道地址（`pi_020`）、未去前缀仓库后缀与招聘口号（`pi_016`/`pi_017`） | `0` 损坏 JSON 行 |
| **Holdout A1 & A2 四阶段单 Hunter** (`33662da`) | 同上（A1 12 条 + A2 9 条） | 四阶段混合流水线（单 Hunter，`24/24` 测试） | **PASS**：Holdout A1 降至 `Low`（`0` Crit/High/Med，2 轮收敛）；**Holdout A2 降至 `None`（全级别 `0` 发现，4 轮收敛）** | `utility_intact: true`；原始 SHA-256 `12/12` 不变 |
| **零污染 Holdout #3 分片多智能体** (`b80590d` **冻结**) | **10 条全新跨项目 Claude Code + Codex 会话（5.76 MB，6 个未接触项目）** | **代码冻结后一次性运行**：6 并行分片 Hunter（`1.15 MB`/片，`115 KB` 重叠）+ 1 Merger (`26/26` 测试) | **FIRST-SHOT PASS（首轮一次性通过）**：Claude (4) 与 Codex (6) 均为 `Low`（**`0` Critical / `0` High / `0` Medium**，仅存留不透明 `req_` ID 与粗粒度时区等 `Low` 项） | `utility_intact: true`；原始 SHA-256 `10/10` 不变；`0` 损坏 JSON 行 |

---

## 复现与接入

```shell
# 1. 获取上游 v0.2.0 技能并运行 26 项全量测试
git clone --branch v0.2.0 --depth 1 https://github.com/linmou/clean-session-curate.git /tmp/clean-session-curate
cd /tmp/clean-session-curate && npm run build

# 2. Pass 1：非交互式确定性 AST 脱敏
node scripts/anonymize_session.mjs --input <raw.jsonl> --output <session-export/harness_001.jsonl> --profile <tmp-profile.json>

# 3. Pass 2：按 max(20%, 20 KB) + 10% 重叠切分语料供并行 Hunter 与 Merger 审阅
node scripts/chunk_for_hunter.mjs --input-dir <session-export> --out-dir /tmp/hunter-shards

# 4. Pass 3：回填 Merger 产出的临时词表并清扫路径
node scripts/anonymize_session.mjs --input <session-export/harness_001.jsonl> --output <session-export/harness_001.jsonl> --terms /tmp/merged-terms.json
node scripts/scrub_export_paths.mjs --root <工作根> --export-dir <session-export>

# 5. Pass 4 盲测验收通过（0 Critical/High）后打包
node scripts/package_export.mjs --root <工作根> --export-dir <session-export> --output <工作根>/session-export.zip
```
