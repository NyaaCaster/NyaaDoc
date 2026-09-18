# NyaaDoc V1 开发计划 SSOT

> **唯一事实来源**。开发阶段的架构、约束、进度、验证口径以本文件为准。
> 进度符号：⬜ 未开始 / 🟡 进行中 / ✅ 已完成
> 最后更新：2026-09-18

---

## 一、目标

建立 **NyaaDoc**——基于 Docmost 的二次开发发行版，交付一个可公网访问的自托管知识库文档站：

- 公开文档**免鉴权**阅读；文档增删改查**全部在 Web 前端**完成
- 每篇文档可**单独控制是否发布**；编辑与管理**必须登录**
- 界面中文化、品牌化为 NyaaDoc、通过 HTTPS 对外提供

V1 的成功判据：**在 macmini 上通过自有域名 HTTPS 访问 NyaaDoc，F1–F8 全部验收通过**。

---

## 二、当前仓库状态

| 项 | 值 |
|---|---|
| 本地开发目录 | `H:\GitHub\NyaaDoc` |
| GitHub 仓库 | `https://github.com/NyaaCaster/NyaaDoc.git`（公有，默认分支 `main`） |
| 远端内容 | ✅ **已清空**（2026-09-18）。原 `README.md`、`docusaurus.config.js` 系该仓库早期 Docusaurus/Netlify 尝试的遗留，**与本知识库项目无关**，已删除（commit `50d5b91`、`8739fa1`）；后续**不得引用、复用或混淆**这些旧内容 |
| 本地 Git | ⬜ **尚未初始化**（P1 处理） |
| 当前阶段 | P0 立项（方案/审核/SSOT 文档已建立） |

---

## 三、已确认架构

### 3.1 上游基座（锁定）

| 项 | 值 |
|---|---|
| 上游项目 | `docmost/docmost` |
| 版本 | **v0.96.0**（release 2026-09-08） |
| 镜像 | `docmost/docmost:0.96.0` |
| digest | `sha256:b56947fcfd08aab8fae12a377e1792784786adbf8b96e4281f14ef4fc072685a` |
| 依赖镜像 | `postgres:18`、`redis:8` |
| 许可 | AGPL-3.0（核心）；`apps/*/src/ee`、`packages/ee` 为独立企业许可 |

### 3.2 三层结构

- **品牌层（本项目）**：`branding/` 下的中文语言包补丁与图标 → 通过薄镜像 COPY 覆盖
- **上游基座（不修改）**：Docmost 官方镜像，按 digest 锁定
- **发行层（本项目）**：`Dockerfile`、`docker-compose*.yml`、`.env`、`rebuild.py`、`restart.py`、nginx 配置、文档

### 3.3 品牌化契约（**本项目的技术核心，改动前必读**）

Docmost 前端使用 `i18next-http-backend`，**运行时**通过 HTTP 加载翻译：

```
浏览器 GET /locales/zh-CN/translation.json   ← 由 apps/client/dist 静态目录提供
```

官方 Dockerfile 将 `apps/client/dist` 整体拷贝进运行镜像，因此**覆盖以下路径即可完成品牌化，无需编译前端**：

| 覆盖目标（容器内） | 源（本仓库） | 作用 |
|---|---|---|
| `/app/apps/client/dist/locales/zh-CN/translation.json` | `branding/locales/zh-CN/translation.json` | 中文文案 + 产品名 |
| `/app/apps/client/dist/icons/favicon-16x16.png` | `branding/icons/favicon-16x16.png` | 站点图标 |
| `/app/apps/client/dist/icons/favicon-32x32.png` | `branding/icons/favicon-32x32.png` | 站点图标 |

> ⚠️ **路径变更即失效**：上游若调整 `dist` 结构，COPY 会失败或覆盖无效。
> 因此 `rebuild.py` 必须**构建期校验**目标路径存在（校验失败即中止构建）。

---

## 四、关键约束

### 4.1 许可与合规

1. 上游核心 **AGPL-3.0**：作为网络服务对外提供时须提供修改后的完整源码（§13）→ 公开仓库满足。
2. **保留上游 `LICENSE` 与版权声明**，显著标注"基于 Docmost 修改"（AGPL §5）。
3. `apps/client/src/ee`、`apps/server/src/ee`、`packages/ee` 的**许可文件不得改动**，不得当 AGPL 分发。
4. 品牌不得暗示与 Docmost 官方关联；上游标识须在品牌层被替换但不得删除版权信息。

### 4.2 安全

1. 应用端口**仅绑定回环**（`127.0.0.1:3300`），公网只经 nginx。
2. `APP_SECRET` ≥32 字符随机值；数据库密码强口令；两者只存 `.env`。
3. 私有镜像仓库地址**禁止硬编码**在任何 Git 跟踪文件中（用 `PRIVATE_DOCKER_REGISTRY_HOST`）。
4. nginx 反代**必须支持 WebSocket**（实时编辑器依赖）。
5. 公开分享 = 互联网可见，**敏感内容一律不得开启分享**。

### 4.3 工程规范

- 自动化脚本一律 **Python 3**（禁止 `.ps1` / `.sh`）。
- 私有仓库地址在脚本输出中掩码为 `<PRIVATE_REGISTRY>`。
- 项目核心文件保留**非注释、运行时可见**的代码签名：`Nyaa be with you.`
  - 落点：`Dockerfile` 的 `LABEL org.nyaadoc.blessing` + `branding/version.json`
- 不提交：`.env`、`.env.*`（`.env.example` 除外）、`node_modules/`、`dist/`、`*.log`、数据库卷、>5MB 二进制。

### 4.4 内容治理（**唯一不可技术补救的风险**）

- 默认全部私有，分享开关由人工逐篇开启。
- 上线前须产出"可公开内容清单"，复核后再开放公网入口。

---

## 五、目录结构约定

```
H:\GitHub\NyaaDoc\
├── .docs\                      # 本文档目录（方案/审核/SSOT/交接）
├── .ref\                       # 外部参考资料（不入 Git）
├── branding\                   # 品牌资产（唯一品牌来源）
│   ├── locales\zh-CN\translation.json
│   ├── icons\favicon-16x16.png
│   ├── icons\favicon-32x32.png
│   └── version.json            # 产品名/版本/代码签名
├── deploy\                     # 部署编排
│   ├── docker-compose.yml          # 本地开发（含 build: .）
│   ├── docker-compose.publish.yml  # 发布（仅引用私有仓库镜像）
│   ├── nginx\                      # macmini 侧 vhost 片段
│   └── .env.example
├── scripts\
│   ├── rebuild.py              # 构建 → 校验 → 推送私有仓库
│   ├── restart.py              # macmini 侧拉取重启
│   ├── verify-branding.py      # 品牌化冒烟验证（R4/R5 缓解）
│   └── backup-db.py            # 定期 pg_dump（待拍板 D10）
├── Dockerfile                  # FROM docmost/docmost:0.96.0 + COPY branding
├── .gitignore
├── .env                        # 不入 Git
├── meta.json                   # 项目元数据单一来源
├── CLAUDE.md
└── README.md
```

---

## 六、环境变量约定

`.env.example` 为公开模板，真实值只写本地 `.env`。

| 变量 | 示例 | 说明 |
|---|---|---|
| `PRIVATE_DOCKER_REGISTRY_HOST` | `<PRIVATE_REGISTRY>` | 私有仓库 host，**只从 `.env` 注入** |
| `NYADOC_HTTP_PORT` | `3300` | 宿主机回环端口 → 容器 3000 |
| `APP_URL` | `https://<YOUR_DOMAIN>:43083` | **必须与实际访问地址一致**，否则邮件/分享链接错误 |
| `APP_SECRET` | `<32+ 随机串>` | 必填，≥32 字符，默认值会导致启动失败 |
| `DATABASE_URL` | `postgresql://nyaadoc:<pwd>@db:5432/nyaadoc` | Postgres 连接串 |
| `REDIS_URL` | `redis://redis:6379` | Redis 连接串 |
| `POSTGRES_PASSWORD` | `<强口令>` | 数据库口令 |
| `STORAGE_DRIVER` | `local` | 文件存储驱动 |
| `MAIL_FROM_NAME` | `NyaaDoc` | 邮件发件人显示名（**品牌化免改码**） |
| `DISABLE_TELEMETRY` | `true` | 关闭上游遥测 |
| `TZ` | `Asia/Shanghai` | 时区 |
| `NYADOC_DATA_DIR` | `H:\..` / macmini `/root/DockerContainer/DockerRes/NyaaDoc` | 持久化根目录 |

**未启用**（企业版）：`AI_*`、`SEARCH_DRIVER=typesense`、SAML/SCIM 相关。

---

## 七、版本与阶段划分（V + P）

### V1 — 薄镜像品牌化上线（当前目标）

| 阶段 | 内容 | 状态 |
|---|---|---|
| **P0** | 立项：方案、审核、SSOT、交接文档 | ✅ 已完成 |
| **P1** | 工程骨架与上游基线：Git 初始化、清空远端占位文件、`.gitignore`、`meta.json`、`CLAUDE.md`、拉取上游镜像并核实静态资源路径 | ⬜ 未开始 |
| **P2** | 品牌资产与薄镜像：抓取并补全 `zh-CN` 语言包、制作图标、`Dockerfile`、`rebuild.py`、构建期路径校验、代码签名落点 | ⬜ 未开始 |
| **P3** | 部署编排与本地验证：compose 双文件、`.env.example`、`restart.py`、`verify-branding.py`、数据库备份脚本（待拍板） | ⬜ 未开始 |
| **P4** | macmini 上线与 HTTPS：推送镜像、部署目录、nginx `43083 ssl`、证书复用、容器不映射数据库端口 | ⬜ 未开始 |
| **P5** | 线上验收与交付：F1–F8 逐条验收、内容治理清单、公网开放、文档收尾 | ⬜ 未开始 |

### V2 — 源码级二开（**待 V1 验收后再决定是否启动**）

| 阶段 | 内容 | 状态 |
|---|---|---|
| **P6** | fork 血缘建立与源码构建链路（fork 名固定为 `docmost`，`upstream` remote + `nyaadoc` 分支策略） | ⬜ 未开始 |
| **P7** | 外观主题源码改造（组件样式、主题色、登录页品牌化） | ⬜ 未开始 |

> V2 不在 V1 范围内，禁止在 V1 期间预先铺开。

---

## 八、当前已完成（P0）

- 完成 Obsidian 生态调研，确认其无法满足 F1/F2，排除静态发布器路线（Quartz / Digital Garden / Perlite / Webpage HTML Export）。
- 完成 8 个自托管知识库方案的横向评测，选定 Docmost。
- 实测确认 Docmost 关键能力：页面级公开分享、单篇 .md 导出、整 Space 层级化 Markdown ZIP 导出、免鉴权公开页、编辑需登录。
- **实测确认品牌化机制的可行性**（`i18next-http-backend` 运行时加载 + `/locales/zh-CN/translation.json` 可访问）。
- 实测确认 `zh-CN` 语言包已存在且覆盖完整（含"公开分享""分享到网页""允许搜索引擎索引页面"等关键文案）。
- 实测确认 macmini 资源与端口可用性：`3300`/`3301`/`43083` 空闲；磁盘 151 GB 可用；内存 11 GB 可用。
- 澄清 `https.conf` 中的 `43083` 仅为注释文字（CLIProxyAPI 实际监听 `48317`），端口可用。
- 完成上游版本核实：最新 release `v0.96.0`，镜像 `0.96.0` = `latest`，digest `b56947fc…`。
- 建立 `.docs/初始设计.md`、`.docs/设计审核.md`、`.docs/开发计划-SSOT.md`、`.docs/阶段交接-001.md`。
- 建立项目级 `CLAUDE.md`（项目规则与八条硬约束）。
- **清空远端 `NyaaCaster/NyaaDoc` 的废弃残留文件**（早期 Docusaurus/Netlify 尝试，与本知识库项目无关；commit `50d5b91`、`8739fa1`），仓库现为 0 文件。

## 九、当前验证结果

P0 为立项阶段，无代码产物；已验证项均为**外部事实核实**，证据记录于《设计审核》第 2.2 节证据链表。

尚未验证（归入 P1–P5）：镜像静态资源路径在**运行容器内**的实际存在性、品牌化覆盖后的渲染效果、nginx WebSocket 透传、端到端免鉴权访问。

## 十、仍需继续的验证

1. **P1**：`docker run` 后进入容器确认 `/app/apps/client/dist/locales/` 与 `icons/` 实际存在。
2. **P2**：覆盖后浏览器实测中文文案与图标生效；`docker inspect` 可见代码签名。
3. **P4**：nginx `43083` WebSocket 透传（实时编辑器不报连接失败）。
4. **P5**：F1–F8 逐条验收，含无痕窗口免鉴权验证与未登录编辑拦截。

## 十一、下一阶段推荐执行顺序（P1）

1. ~~清空 `NyaaCaster/NyaaDoc` 远端废弃残留文件~~ ✅ 已完成（2026-09-18，仓库现为 0 文件）
2. ~~建立项目级 `CLAUDE.md`~~ ✅ 已完成
3. 建立 `.gitignore`（Python/Node/Docker/`.env`/`.ref` 全覆盖）
4. 建立 `meta.json`（项目名、仓库、端口、镜像名、上游版本的单一来源）
5. `docker pull docmost/docmost:0.96.0` 并进容器核实静态资源路径（**为 P2 解锁**）
6. 本地 `git init` + 关联远端 + 首次提交（`CLAUDE.md`、`.gitignore`、`meta.json`、`.docs/` 文档）

## 十二、验证命令参考

```bash
# 上游镜像与静态资源路径核实（P1）
docker pull docmost/docmost:0.96.0
docker run --rm docmost/docmost:0.96.0 ls -la /app/apps/client/dist/locales/
docker run --rm docmost/docmost:0.96.0 ls -la /app/apps/client/dist/icons/

# 品牌资产与原包比对（P2）
python scripts/verify-branding.py

# 镜像构建与推送（P2/P4）
python scripts/rebuild.py

# macmini 部署（P4）
ssh macmini 'cd /root/DockerContainer/NyaaDoc && python3 restart.py'

# 线上验收（P5）
curl -sk -o /dev/null -w "%{http_code}" https://<YOUR_DOMAIN>:43083/ -H 'Host: <YOUR_DOMAIN>'
git status --short
```

## 十三、提交规范

- Conventional Commits（英文小写）：`feat:` `fix:` `docs:` `chore:` `build:` `refactor:`
- **禁止** `Co-Authored-By` 行；**禁止** `git add -A` / `git add .`（逐文件显式添加）
- 禁止 force push、`--amend` 已推送提交、`--no-verify`、`reset --hard`
- 推送前必做 `git status --short` + staged diff 的 secret 扫描
- 推送走 PAT 临时鉴权（`$GITHUB_PAT`），remote 保持干净 URL
