# NyaaDoc — Claude Code 项目规则

## 项目概述

NyaaDoc 是**基于 Docmost 的二次开发发行版**：一个可公网访问的自托管知识库文档站。

- 上游基座：Docmost v0.96.0（AGPL-3.0，按 digest 锁定，**不修改**）
- 品牌层：中文语言包补丁 + 图标 + 产品名（`branding/`，本项目）
- 发行层：Dockerfile / compose / 部署脚本 / nginx 接入（本项目）
- 仓库：`https://github.com/NyaaCaster/NyaaDoc.git`（主分支 `main`）
- 核心能力：公开文档免鉴权阅读；Web 端增删改查；单篇发布开关；编辑需登录

## 交流语言

默认始终以**简体中文**与用户交流，包括解释、总结、提问、错误说明。代码、标识符、命令行参数、提交信息按惯例使用英文。

## 会话启动必读

每次会话开始前，必须阅读：

1. `.docs/开发计划-SSOT.md` — **唯一事实来源**（架构、约束、进度、验收口径）
2. `.docs/设计审核.md` — 关键决策点 D1–D10、风险 R1–R8
3. `.docs/初始设计.md` — 需求与验收口径 F1–F8
4. 最新的 `.docs/阶段交接-XXX.md` — 精确当前状态与续接提示词

---

## 硬约束（MUST / 不得违背）

### 1. 范围约束

- **V1 只做"薄镜像品牌化"**：`FROM docmost/docmost:0.96.0` + 覆盖静态资源，**不编译前端**。
- **源码级外观改造属于 V2**（P6/P7），**禁止在 V1 期间预先铺开**。
- 不接受"顺手优化"式的范围蔓延；新增需求先走设计审核并更新 SSOT。

### 2. 上游关系

- 上游按 **v0.96.0 + digest `sha256:b56947fc…`** 锁定，**不使用 `latest`**。
- GitHub **不允许重命名 fork**，因此"fork 后改名 NyaaDoc"不可行；代码血缘与品牌发行分离。
- 上游若有安全更新，升级前必须跑品牌化冒烟验证。

### 3. 许可合规

- 上游核心为 **AGPL-3.0**：作为网络服务提供时须提供修改后的完整源码（§13）。
- **保留上游 `LICENSE` 与版权声明**，显著标注"基于 Docmost 修改"（AGPL §5）。
- `apps/client/src/ee`、`apps/server/src/ee`、`packages/ee` 为**独立企业许可**：
  **不得改动其许可文件、不得当作 AGPL 分发、不启用其功能**。
- 品牌不得暗示与 Docmost 官方存在关联。

### 4. 安全

- 应用端口**仅绑定回环**（`127.0.0.1:3300`），公网只经 nginx `43083 ssl`。
- `APP_SECRET` ≥32 字符随机值，只存 `.env`；数据库强口令。
- 私有镜像仓库地址**禁止硬编码**在任何 Git 跟踪文件中，只从 `.env` 的
  `PRIVATE_DOCKER_REGISTRY_HOST` 注入，脚本输出中掩码为 `<PRIVATE_REGISTRY>`。
- nginx 反代**必须支持 WebSocket**（实时编辑器依赖）。
- **内容治理**：公开分享 = 互联网可见，敏感内容一律不得开启分享开关。

### 5. 品牌化契约（改动前必读）

Docmost 前端使用 `i18next-http-backend` **运行时**加载翻译，因此覆盖静态文件即可品牌化：

| 覆盖目标（容器内） | 源（本仓库） |
|---|---|
| `/app/apps/client/dist/locales/zh-CN/translation.json` | `branding/locales/zh-CN/translation.json` |
| `/app/apps/client/dist/icons/favicon-16x16.png` | `branding/icons/favicon-16x16.png` |
| `/app/apps/client/dist/icons/favicon-32x32.png` | `branding/icons/favicon-32x32.png` |

- **上游若调整 `dist` 结构，覆盖即失效** → `rebuild.py` 必须做**构建期路径校验**，缺失即中止构建。
- 每次上游升级后必须运行 `scripts/verify-branding.py` 冒烟验证。

### 6. 代码签名

核心文件保留**非注释、运行时可见**的代码签名：`Nyaa be with you.`

- 落点：`Dockerfile` 的 `LABEL org.nyaadoc.blessing` + `branding/version.json`
- 禁止：写为 `//` 注释、渲染到用户可见 UI 文本

### 7. 不提交文件

`.env`、`.env.*`（`.env.example` 除外）、`.ref/`、`node_modules/`、`dist/`、`*.log`、
数据库卷、>5MB 二进制、`.claude/settings.local.json`。

### 8. 废弃内容隔离

仓库过往的 Docusaurus/Netlify 遗留内容（`README.md`、`docusaurus.config.js`）**已于 2026-09-18 删除**，
与本知识库项目**无任何关系**，**不得引用、复用或从中派生设计**。

---

## 自动化脚本规范

- 一律使用 **Python 3**（`rebuild.py` / `restart.py` / 验证脚本），**禁止 `.ps1` / `.sh`**。
- 现有 `.ps1`/`.sh` 应迁移为 `.py`。

## Docker 规范

| 文件 | 用途 | 运行位置 |
|---|---|---|
| `scripts/rebuild.py` | 构建 → **路径校验** → 推送私有仓库 | Windows 开发机 |
| `scripts/restart.py` | 拉取 → 重启容器 → 清理悬空镜像 | macmini |
| `deploy/docker-compose.yml` | 本地开发（含 `build: .`） | Windows |
| `deploy/docker-compose.publish.yml` | 发布（仅引用 `${PRIVATE_DOCKER_REGISTRY_HOST:?err}/nyaadoc:latest`） | macmini |

- 私有仓库地址规则：Windows 本地构建推送与 macmini 部署拉取使用**不同 host**（规避 NAT hairpin），
  二者**都只从 `.env` 的 `PRIVATE_DOCKER_REGISTRY_HOST` 注入**；
  **具体地址禁止写入任何 Git 跟踪文件**，脚本输出中一律掩码为 `<PRIVATE_REGISTRY>`。
- **禁止未经用户明确要求启动容器**（`docker compose up` / `docker run` / `docker start`）。
  允许：`docker build`、`docker push`、`docker compose down`、`docker images`、`docker compose ps`。
- 部署目录：macmini `/root/DockerContainer/NyaaDoc/`；持久化 `/root/DockerContainer/DockerRes/NyaaDoc/`。

## Git 提交规范

- Conventional Commits（英文小写）：`feat:` `fix:` `docs:` `chore:` `build:` `refactor:`
- **禁止** `Co-Authored-By` 行
- 始终 `git add <file>` 显式添加；**禁止** `git add -A` / `git add .` / `git add -u`
- 禁止：force push、`--amend` 已推送提交、`--no-verify`、`git rebase`、`reset --hard`
- 推送前必须 `git status --short` + staged diff 的 secret 扫描
- 推送走 `$GITHUB_PAT` 临时鉴权（`insteadOf` 重写），**remote 保持干净 URL，绝不持久化 token**
- 多行 commit message 使用 HEREDOC

## Vibo Coding 工作规范（简版）

```
初始设计 → 设计审核 → 关键决策拍板 → SSOT 计划 → 分 P 推进 → P 阶段收尾
```

- **V（版本）**：V1 薄镜像品牌化上线；V2 源码级二开（待 V1 验收后决定）。
- **P（阶段）**：V1 = P0 立项 / P1 工程骨架 / P2 品牌资产与薄镜像 / P3 部署编排 / P4 macmini 上线 / P5 线上验收。
- 每个 P 必须**可独立验证、可独立提交**；不出现"半个 P 交完等下个补完"。
- 状态符号：⬜ 未开始 / 🟡 进行中 / ✅ 已完成（维护于 SSOT 的 V+P 表）。
- **实现工作以 plan 模式推进**：先明确方案、涉及文件、验证步骤，获批后执行。
- **每 P 收尾必做**：① 更新 SSOT 状态与变更；② Git 提交推送；③ 新建/更新 `.docs/阶段交接-XXX.md`（含"续接提示词"）。

### 交接文档的"续接提示词"

一段可直接粘贴给新对话的提示词，要求：以 plan 模式推进、列出必读文档、说明当前进度、
指出下一阶段第一件事、重申关键约束，长度约 10–20 行。

### 跨对话接续

用户说"继续开发 NyaaDoc"时：读 `CLAUDE.md` → 读 SSOT → 读最新交接文档 → 按"续接提示词"进入下一 P。
