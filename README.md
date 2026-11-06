# Large-Lunch-Model

百模百味，寻找你最喜欢的赛博饭友。

面向悉尼 CBD 及周边的门店和菜品目录，以可追溯的菜单事实支持预算内选餐，逐步扩展不同 Model、Persona 与 Skill。

## 2026-11-06

这次新增 Sprint 1 的 MVP、两条用户流程和 18 条 User Stories，并把注册登录、邮箱验证、Cloudflare 防护、AI 配额和跨实例并发写进技术基线、数据、AWS、本地开发和交付文档。部署方案仍是 **Next.js + Java + Python、AWS ECS Fargate**。服务、数据库、Compose、测试、AWS / Cloudflare 资源和平台接入都还没有实现。文档里的命令、API 和目录是初始化约定。

开发方式按一人开发、AI 辅助实现与 review 来排。本轮目标包含浏览和聊天两条路径，在确认每周投入之前不设日期，也不能直接承诺两周全部完成。

用户已确认的预算规则：**按菜品小计计算，包含所选规格、必选加料、实际选择的可选加料和数量；配送费与平台 service fee 等订单级费用另列。** 页面使用“菜品小计，配送费及平台费用另计”。持久化口径是 `MENU_SUBTOTAL`，首版货币为 `AUD`。

首轮访问规则：

- 匿名用户可浏览、搜索、查看详情和跳转地图 / 外卖平台。新的付费 AI chat、推荐和概览生成要求登录、邮箱验证、有效账号和 Turnstile 服务端验证。
- MySQL 统一执行跨实例 admission，限制同一用户多 tab、全局与 provider 并发，并预留每日 request / token quota。重试、取消、重启或扩容后额度仍然有效。
- 云端路径是 Cloudflare → 受限 ALB → ECS Fargate。Cognito 身份入口使用独立 AWS WAF。并发和额度数值是可配置初值，要在 load test 之后调整。
- 私人会话按 owner 保存并隔离。菜单 provenance、版本复核、审核发布和费用口径继续沿用。

## 文档导航

| 文件 | 解决的问题 |
| --- | --- |
| [MVP 与 User Stories](docs/sprint-1-mvp-and-user-stories.md) | 两条用户流程、S1-US01–18、安全 / 并发 enablers、单人开发顺序与验收 |
| [技术栈与 architecture 要求](tech-stack-requirements.md) | 阶段范围、服务边界、预算、auth / admission、API 与验收 |
| [数据要求](docs/data-requirements.md) | 来源权限、菜单版本、账号 / 私人会话、quota ledger、导入审核发布与质量 |
| [AWS 要求](docs/aws-requirements.md) | Fargate、Cloudflare / Cognito 防护、origin、IAM、存储、CI/CD、成本与恢复 |
| [本地开发要求](docs/local-development.md) | Host + Docker Compose、auth / Turnstile mock、fixture、多实例验收、启动与排障 |
| [交付实践与团队组织](docs/delivery-model-comparison-au-us-cn.md) | 团队 ownership、Discovery / Alpha / Beta、Sprint 与交付协作建议 |

先读 MVP 与 User Stories，再读总体 architecture 和三份专题。User Stories 定义用户体验；技术基线第 5、7 节定义统一 auth、admission 和 API contract；数据文档定义 schema；AWS 与本地文档规定各环境怎么执行和验收。相关 API、fixtures 和验收要一起改。

## 按文档的开发顺序

下面的 A–F 来自 Sprint 1 文档第 8.1 节：先让一条真实浏览路径可访问，再补查询，然后才接登录和固定模型。技术基线第 1.1 节仍划分启动准备、本地 walking skeleton、受控 Alpha、后续增量和 Production / Beta。团队人数、Sprint 长度和上线时间尚未确定。

```mermaid
flowchart TD
    a["A：最早部署。真实 URL 上选择 mode，打开一条门店和菜品并返回"]
    b["B：浏览闭环。搜索、门店、菜品、返回、导航或外卖链接"]
    c["C：查询能力。类型、菜系、距离和组合条件"]
    d["D：AI 访问门槛。验证登录、Turnstile、owner 隔离、共享额度，先用 mock"]
    e["E：固定 AI。一个 provider 的门店概览和聊天，经 Java 复核"]
    f["F：试用发布。两条 E2E、真实 provider smoke、受保护 HTTPS"]
    a --> b --> c --> d --> e --> f
```

匿名浏览可以先闭合。付费 AI 要等身份、Turnstile、owner 隔离和并发 / 额度控制一起通过，再打开真实 endpoint。

```mermaid
flowchart LR
    home["首页"] --> mode["选择堂食或外卖"]
    mode --> search["搜索门店或菜品"]
    search --> branch["门店详情"]
    branch --> item["菜品详情"]
    item --> branch
    branch --> out["地图导航或已核验的外卖平台页"]
```

```mermaid
flowchart LR
    chat["进入 AI 聊天"] --> login["注册、确认邮箱并登录"]
    login --> send["用户主动发送"]
    send --> gate["Turnstile 与 Java 身份 / 配额 / 并发校验"]
    gate --> model["一个 provider，Java 复核后的推荐卡片"]
    model --> detail["进入与浏览相同的门店和菜品详情"]
```

首轮是单人、单店、单渠道选餐，使用一个 provider 和一个 model。门店概览和聊天是同一 AI 服务的两个任务，页面不提供 model、Persona 或 Skill 切换。样本仍按可审核质量准备，规划量级是 5–10 家门店、30–50 个 MenuItems，尚未采集。多模型比较、收藏、多人点菜、站内下单、自动 OCR、大规模 crawler、embedding、复杂 LangGraph、Live2D 和 3D 留在后续 backlog。

图使用 Markdown 的 `mermaid` 代码块。GitHub 和 Cursor 预览会直接渲染。
