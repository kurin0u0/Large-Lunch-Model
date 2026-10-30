# Large-Lunch-Model：数据要求

- 版本：v0.1；更新日期：2026-10-29（Australia/Sydney）。
- 状态：待实现的 data contract 与验收基线；本文未创建数据库、采集数据或取得平台授权。
- 适用范围：悉尼 CBD 及周边的门店、菜单、渠道报价、来源、审核、发布和推荐证据。
- 用户已确认：预算按菜品小计，包含所选规格与必选加料；配送费和平台 service fee 另列，不计入预算。
- 关联文件：[技术架构](../tech-stack-requirements.md)、[AWS 要求](aws-requirements.md)、[本地开发](local-development.md)。

## 1. 首阶段交付范围

首轮准备 5–10 家门店、30–50 个 MenuItems，数量是起步规模建议，不代表已有数据。每个可推荐项至少有一个经过审核、价格明确且未过期的 OfferVersion。

首个数据闭环是：确认来源权限 → 获取少量样本 → schema validation → 人工审核 → Java 发布 → 查询候选 → 推荐结果复核。人工整理的 CSV / JSON 可以完成首轮 ingestion；审核可以由受控 CLI 调用 Java Admin API 完成，完整管理界面、OCR、四平台全量抓取均不构成首轮 dependency。

范围内包含门店、菜品、确定规格、价格、来源、更新时间、可用性以及有证据的饮食属性。订单创建、支付、配送运力和实时 checkout quote 不在当前实现范围。平台菜单接入仅在权限与数据用途已经确认后启用。

## 2. 预算与价格 contract

### 2.1 已确定的口径

HTTP request 使用 `budget.scope`，持久化对应 `budget_scope`，固定为 `MENU_SUBTOTAL`，首版 currency 固定为 `AUD`。预算为一次推荐方案的菜品总小计，不隐含“人均预算”。首轮只启用单人午餐；后续多人场景继续使用整份方案的总预算，须另行定义份量与共享规则。

```text
line_subtotal = quantity × (selected_variant_price + sum(selected_option_delta))
menu_subtotal = sum(line_subtotal)
budget_pass = menu_subtotal <= budget_amount
```

- `selected_variant_price` 对应确定规格，例如普通份或大份；同一规格的基础价只计一次。
- 所有必选 option groups 必须完成合法选择；已选 optional add-ons 同样计入菜品小计。
- `selected_option_delta` 为每个已选选项相对该规格基础价的调整额，不能同时累加“整份价格”和同一规格的替换价格。
- 菜品金额要求有明确的含税口径。若来源只给未税价格且无法确定含税金额，该 Offer 不进入严格预算候选。
- 配送费、平台 service fee、small-order fee、订单级包装费等单独列为 excluded fees。嵌入菜品标价中的费用不擅自扣除。
- 未知费用使用 `null` 与 `UNKNOWN`，不能用 `0.00` 表示未获取。
- 首版不依赖会员价、用户专属 coupon 或未验证的优惠；条件不确定时使用已确认的普通标价。
- 页面显示“菜品小计，配送费及平台费用另计”，不将预算通过描述为 checkout total 保证。

首轮为单人、单店、单渠道推荐，一个方案只组合相同 Branch、channel 和 fulfillment 的菜品；不同平台、门店或售卖方式是独立候选方案，不能拼成一个不可实际购买的低价组合。该限制是本版设计默认，后续跨店点餐需另行定义。

### 2.2 金额与序列化

MySQL 金额使用 `DECIMAL(12,2)`，Java 使用 `BigDecimal`，Python 使用 `Decimal`；HTTP JSON 使用带两位小数的字符串，如 `"18.50"`。quantity 为正整数。禁止把金额经过 JavaScript / Python binary float 运算后写回。

首版只接受两位小数 AUD，输入超出精度或非 AUD 时返回 validation error；不做隐式汇率转换。数据库约束禁止负基础价，option adjustment 可为负，但最终 line subtotal 不得为负。

推荐请求示例，所有 ID 均为说明用 placeholder：

```json
{
  "budget": {"amount": "25.00", "currency": "AUD", "scope": "MENU_SUBTOTAL"},
  "party_size": 1,
  "fulfillment_type": "PICKUP",
  "dietary_constraints": [],
  "persona_id": "persona_default",
  "skill_id": "budget_lunch"
}
```

预算必须由 Java 在候选筛选及最终结果两个位置计算。Python 返回所选 ID、数量与选项，金额由 Java 关联相同版本的业务记录重新计算。

## 3. 来源接入与使用权限

### 3.1 来源优先级

| 来源 | 可承担的用途 | 接入条件与已核实边界 |
| --- | --- | --- |
| 商家直接提供的菜单或 feed | 首批 menu facts 与后续更新 | 记录商家授权范围、联系人或授权证据、更新责任；商家也须有权提供相应图片与内容 |
| 商家官网菜单 | 有限范围内的菜单采集 | 逐来源确认获取、存储、展示及 AI processing 条件；公开可访问不自动等于可无限复制 |
| Google Maps / Places API | 门店发现、外部 ID、获准的详情展示 | Places API 限制 caching / storage；place ID 可长期保存，不代表完整 response 可长期保存。[官方 policy](https://developers.google.com/maps/documentation/places/web-service/policies) |
| Uber Eats | 获授权门店的菜单与平台报价 | Marketplace API 需要相应批准和 scopes；商家通过 OAuth 授权接入门店，不提供可任意读取所有门店的默认权限。[官方 authentication](https://developer.uber.com/docs/eats/guides/authentication) |
| DoorDash | 获授权 integration 的菜单与平台报价 | Marketplace API 属于 limited access，须申请批准。[官方说明](https://developer.doordash.com/en-US/docs/marketplace/overview/getting_started/setting_up_account/) |
| HungryPanda | 获授权的商家 / partner 数据 | 本次未确认公开的全平台查询 API；官网列有 merchant 与 POS partners，实际访问范围需单独确认。[官方 merchant 页面](https://www.hungrypanda.co/merchants/) |

平台资料核对日期为 2026-10-28。最终使用范围以项目实际签署的协议及启用时的官方条款为准。接入某平台不是首轮必须满足的条件。

### 3.2 Source registry

每个来源必须先登记 `Source`，至少包含：

| 字段 | 要求 |
| --- | --- |
| `source_id`, `source_type`, `provider` | 内部稳定 ID；类型如 MERCHANT_UPLOAD、MERCHANT_WEBSITE、PARTNER_API |
| `terms_url`, `terms_checked_at`, `permission_evidence_ref` | 官方条件及核对日期；授权证据使用受限对象引用 |
| `fetch_allowed`, `storage_allowed`, `display_allowed`, `llm_processing_allowed` | 按用途分别记录 ALLOWED、DENIED、UNKNOWN，不能用一个 approved 代表全部权限 |
| `retention_policy`, `expires_at` | 可保留的字段、原始对象期限、衍生数据期限；与实际协议一致 |
| `attribution_rules`, `redistribution_allowed` | UI attribution 与样本是否可进入 Git / 导出包分别判断 |
| `allowed_fields`, `redacted_fields` | 可采集、可保存、可展示及可送入模型的 field allowlist |
| `refresh_policy`, `owner`, `status` | 更新频率、维护人；ACTIVE、PAUSED、REVOKED |

涉及某用途的权限为 UNKNOWN 时，不执行该用途。允许在 UI 展示不代表允许送到外部 LLM；允许 API 获取不代表允许在 S3 保存完整 response。清洗、翻译、embedding 或截图不会自动改变来源使用条件。

### 3.3 存储规则必须作用于整个 pipeline

- 仅允许 transient processing 的内容不进入 raw bucket、持久化 request log、队列消息、prompt tracing 或备份副本；如不能满足处理边界，停用该来源路径。
- 允许限时存储的内容必须有 `delete_after`，原始文件、结构化副本、cache、日志和 versioned object 按同一政策处理。
- 来源撤销或授权到期时阻止新推荐与新展示，并执行规定的删除或去标识流程；保留合规允许的最小 audit metadata。
- 推荐复现受保留条件约束。若证据必须删除，保留允许的 ID、版本、删除原因，并标记 `evidence_unavailable`，不承诺永久完整复现。
- 不能证明可再分发的数据不进入 Git fixtures；使用明确授权样本或自行构造并标明 synthetic 的测试数据。

## 4. Canonical data model

### 4.1 目录与报价实体

| Entity | 含义与关键字段 |
| --- | --- |
| Restaurant | 品牌或经营主体；`restaurant_id`, `name`；单店也可以有一个 Restaurant |
| Branch | 实际门店；`branch_id`, `restaurant_id`, 地址、IANA timezone、坐标及其来源、营业时间 |
| Dish | 可选的通用菜品分类概念；用于关联检索，不负责存门店价格；首轮可延后丰富 |
| MenuItem | 门店菜品 identity；`menu_item_id`, `branch_id`, 原文名称、可选译名、description、可选 `dish_id` |
| Offer | 某 MenuItem 在某 channel、fulfillment、具体规格下的可售 identity；`offer_id`, `menu_item_id`, `channel`, `fulfillment_type`, `variant_key` |
| OfferVersion | 不可原地修改的报价事实；`offer_version_id`, `offer_id`, base price、option groups、currency、证据引用、observed time、available state |
| MenuSnapshot | 一个 Branch + channel + fulfillment 范围内发布的 immutable manifest，引用确定的 MenuItem revision 与 OfferVersion IDs |
| ExternalMapping | 外部店铺 / 菜品 / 报价 ID 到内部 identity 的映射、confidence、review 状态与证据 |

`channel` 使用 `MERCHANT_DIRECT`、`UBER_EATS`、`DOORDASH`、`HUNGRYPANDA` 等稳定代码；`fulfillment_type` 为 `DINE_IN`、`PICKUP`、`DELIVERY`。DELIVERY 场景也使用菜品小计预算，并展示费用排除提示。

MenuItem 的可变描述、饮食事实等通过具有 `menu_item_revision_id` 的 immutable `MenuItemRevision` 保存。MenuSnapshot 必须冻结描述与饮食事实版本，不能只冻结价格并让历史推荐读到今天的属性。

Option groups 作为 OfferVersion 的 immutable 内容，至少包含 `option_group_id`、`min_select`、`max_select`、允许组合规则。每个 option 包含 `option_id`、`price_delta`、availability、dietary / allergen evidence 及来源引用。必选与已选 optional options 必须满足组内约束；首轮同一 option 最多选择一次，option 计数不与菜品 quantity 混用。若允许替换配料，只有证据明确说明移除原配料时才能改变其 allergen 判定；否则按 base、所选规格与所有已选 options 的组合证据判断。含乳酱料不能因为菜品本体标注不含乳而绕过禁忌。

OfferVersion 不代表实时库存保证。`availability` 为 AVAILABLE、UNAVAILABLE、UNKNOWN；严格候选只使用有当前有效证据的 AVAILABLE。来源标记“菜单上有此项”与现场库存确认的证据类型须区分展示。

### 4.2 来源、批次与审计实体

| Entity | 含义与关键字段 |
| --- | --- |
| SourceRecord | 一次来源观察；`source_record_id`, `source_id`, `external_id` / URL、`observed_at`, `received_at`, content hash、允许保存时的 object ref、parser version |
| FieldEvidence | 字段与来源关联；entity revision、field path、`source_record_id`、定位信息、提取方式、confidence、reviewer |
| IngestionBatch | 一次受控导入；`batch_id`, `source_id`, scope、idempotency key、schema version、状态、计数与 error report |
| ReviewRecord | 审核的 actor、时间、决定、理由、审核内容 hash；内容变化即使原审核失效 |
| PublicationRecord | 发布的 snapshot、前后版本、actor、时间、reason、catalog revision |
| OutboxEvent（按需） | 启用主动删除 / 通知时，与发布同一 transaction 写入的后续事件，支持可靠重试 |

内部 ID 使用不可携带业务含义的稳定字符串，例如 UUID；外部 ID 原样作为字符串保存，不能假定数值范围或跨平台唯一。

### 4.3 Field-level provenance

同一 MenuItem 的名称、价格、地址和饮食属性可能来自不同来源，因此每个用于业务判断或展示的事实字段必须关联 FieldEvidence。价格证据必须对应具体 OfferVersion 和规格；不能用门店首页来源代替某个价格的证据。

`source_ids` 在推荐 contract 中明确指向可授权展示的 `SourceRecord` IDs，另保留 `source_id` 表示 registry 来源；两者不能混用。原始资料为私有时，API 返回经许可的来源说明或 URL，不暴露 raw S3 object key、presigned URL 或授权证据。

推断得到的菜系、翻译或口味标签标记 `evidence_type=INFERRED` 并保存方法 / model version。AI 推断不升级为商家确认的过敏原、价格或营业事实。

## 5. Schema 与 data quality

### 5.1 最小必填项

发布一个可推荐 OfferVersion 至少需要：有效 Source、SourceRecord、Branch、MenuItem revision、Offer identity、确定规格、含税 AUD price、option group 规则、observed time、reviewer、有效期和 availability evidence。缺少必填项保留 draft，不以空字符串、零金额或虚构值补齐。

Branch 坐标缺失时仍可进入不含距离条件的目录，但不能声称满足距离过滤。MySQL 地理查询固定 SRID / axis convention；API 使用显式 `latitude` 与 `longitude`，范围分别为 [-90, 90]、[-180, 180]；必须以悉尼已知坐标做空间查询验证。

价格、时间、外键、currency、option group 最小 / 最大选择数和引用版本均由 Java schema validation 检查。所有 timestamp 保存 UTC ISO 8601；营业时间以门店 `Australia/Sydney` 等 IANA timezone 解释，覆盖夏令时和跨午夜营业。

### 5.2 Dedupe 与 entity resolution

- 同名品牌的不同地址保持不同 Branch；仅靠名称相同不能合并。
- 同店同名菜在不同渠道可能是不同份量或套餐；先保留独立 Offer，再审核到 canonical MenuItem 的映射。
- 菜名归一化只作为 candidate matching 的信号，同时比对门店、规格、描述和来源。
- 外部映射以 provider + external entity type + external ID + 必要的店铺 scope 唯一；匹配不明确时使用 UNRESOLVED。
- 人工 merge 必须记录原因和旧 ID alias，不能使历史推荐的引用失效。
- 同一 operation / resource scope 内的 idempotency key 重试不能重复创建批次或版本；新一次真实观察即使内容未变，也应保存允许保留的 observation evidence，以支持 freshness 续期。

### 5.3 饮食禁忌与未知值

饮食 / allergen fact 使用 CONFIRMED_PRESENT、CONFIRMED_ABSENT、UNKNOWN，并关联证据。未标注为 UNKNOWN。

用户设置 hard dietary constraint 时，只纳入满足所需明确证据的候选；UNKNOWN 不能作为“安全”结果。若候选全部被排除，返回无候选及原因，并允许用户自行调整条件；系统不能静默降级为偏好。

## 6. Lifecycle、freshness 与更新

### 6.1 审核发布与新鲜度分离

`workflow_state` 使用 DRAFT → REVIEWED → PUBLISHED；拒绝或修改后回到 DRAFT，撤回为 WITHDRAWN。`freshness_state` 独立计算为 FRESH、STALE、UNKNOWN，不能把 stale 当作审核步骤。

发布不会重置 `observed_at`。新观察可以产生新证据与新版本；修改已审核 payload 后必须重新审核。已发布 snapshot immutable，新数据通过新 snapshot 替换 current pointer。

首轮只有 `PUBLISHED + FRESH + AVAILABLE` 且来源用途允许的版本进入严格推荐。WITHDRAWN、STALE、UNKNOWN 和过期授权版本仍可按保留条件供内部追溯，但不能通过 cache 重新进入候选。

### 6.2 初始 freshness policy

下列是本项目可配置的起步默认值，属于设计选择，不是平台允许存储时间或实时准确性承诺：

| 数据 | 起步 refresh / freshness 上限 | 过期处理 |
| --- | --- | --- |
| 商家直接提供 / 官网菜单的菜品价格 | 7 天 | 排除严格预算推荐，进入重核队列 |
| Partner 平台菜单标价 | 24 小时 | 排除严格预算推荐，重新查询获授权来源 |
| 动态售罄 / 可售状态 | 15 分钟，仅适用于实际提供该动态信号的来源 | 不继续声称当前库存确认；无法证明可售时按 UNKNOWN |
| 营业时间 | 7 天；特别营业日另核实 | 不声称当前营业，保留可用的普通目录信息 |
| 门店地址与稳定身份 | 90 天 | 进入人工复核，距离条件按有效证据决定 |

普通静态菜单可售证据随其菜单 observation 有效期处理，UI 说明“菜单列出，实际售罄以商家为准”；不能伪装成上表的 15 分钟动态确认。

实际有效期取来源许可期限、来源指定有效期和业务 freshness 上限中的最早值。未运行刷新任务时也由查询时的时间判断排除过期数据，不能依赖 scheduler 才改变 freshness。

更新任务先比较 payload / content hash，再产生相应版本。删除菜品、涨价、门店关闭或来源撤销都必须影响 current snapshot 和后续候选；保留旧版本仅供允许的历史追溯。

## 7. Storage 与访问边界

| 数据 | 本地 | AWS | Owner / 约束 |
| --- | --- | --- | --- |
| 获准保存的原始菜单、照片、PDF、response | gitignored `data/raw/` | Private S3 raw bucket / prefix | 来源 policy 决定保留时间，不走公共 CDN |
| 清洗中间产物与 error report | gitignored `data/processed/` | Private S3 processing prefix | 不作为发布目录的唯一事实来源 |
| Source、draft、review、published facts、版本、推荐记录、outbox | MySQL persistent volume | RDS for MySQL | Java 持有业务写权限 |
| 热门查询与短期状态 | Redis | ElastiCache | 可重建，设 TTL；不替代 MySQL |
| 可再分发的小型 fixture、schema、migration | Git；fixture 放在 `data/fixtures/` | CI / release artifacts | 明确授权或 synthetic；不提交批量真实 raw data |

S3 保存大文件，MySQL 保存 metadata、关系和 object reference；不把大段 HTML / 图片 base64 塞入业务表或 SQS。原始文件先进入 quarantine，完成大小、类型、格式与必要的恶意文件检查后才能提取；文件名和来源 URL 不直接用作可执行路径。

Python AI service 不直接访问业务数据库或 raw bucket。Python ingestion worker 有受限对象读取权限，并通过独立 service identity 调用 Java ingestion API；它没有审核、发布或任意 SQL 权限。AI 的只读 tools 与 ingestion 的写入 credentials / scopes 分开。

数据库 backup retention、S3 versioning / lifecycle、日志与 tracing retention 必须服从 Source policy。对象删除不等于 noncurrent versions 或备份中的副本立即消失；不能满足删除时限的来源不进入该长期存储路径，详见 [AWS 要求](aws-requirements.md)。

## 8. Ingestion 与发布 contract

### 8.1 角色与 API

下表均为待实现的接口要求，不代表 repository 已有 endpoint：

| API | Caller / scope | 行为 |
| --- | --- | --- |
| `POST /api/v1/admin/sources` | operator / `catalog:source:manage` | 登记来源 policy、授权证据与用途；ingestion token 无权把权限改为 ALLOWED |
| `POST /api/v1/admin/restaurants`、`POST /api/v1/admin/branches` | operator / `catalog:manage` | 建立品牌、门店及稳定 ID，外部映射保留审核依据 |
| `PATCH /api/v1/admin/sources/{id}` | operator / `catalog:source:manage` | 更新权限、期限或撤销来源；expected revision 防并发覆盖，递增 catalog revision |
| `POST /api/v1/admin/menu-snapshots/{id}/withdraw` | publisher / `catalog:publish` | 原子撤回并清除相应 active pointer、记录原因与 revision；不自动恢复未经重新校验的旧菜单 |
| `POST /internal/v1/ingestion/source-records` | ingestion service / `ingestion:write` | 登记观察时间、URL / 已验证 object reference、hash 与证据定位；校验已存在的 Source policy |
| `POST /internal/v1/ingestion/batches` | ingestion service / `ingestion:write` | 登记来源、批次 scope、schema version 和 idempotency key |
| `POST /internal/v1/ingestion/batches/{id}/records` | ingestion service / `ingestion:write` | 上传有大小上限的结构化 draft records；执行 validation |
| `POST /internal/v1/ingestion/batches/{id}/complete` | ingestion service / `ingestion:write` | 校验 expected record count / manifest hash，固定批次内容后才允许 review |
| `POST /api/v1/admin/ingestion/source-records`、`POST /api/v1/admin/ingestion/batches`、`POST /api/v1/admin/ingestion/batches/{id}/records`、`POST /api/v1/admin/ingestion/batches/{id}/complete` | operator / `catalog:import` | 人工 CLI 的用户身份入口，调用与 internal endpoints 相同的 Java application service；不接受 ingestion service token 代替用户身份 |
| `GET /api/v1/admin/ingestion/batches/{id}` | reviewer / `catalog:review` | 查看数量、validation errors、来源与版本差异 |
| `POST /api/v1/admin/ingestion/batches/{id}/review` | reviewer / `catalog:review` | 对指定 payload hash 审核；保存 actor 与决定 |
| `POST /api/v1/admin/ingestion/batches/{id}/publish` | publisher / `catalog:publish` | 校验版本与权限，在短 transaction 中发布 |

首轮人工 import CLI 放在 `tools/ingestion/`，使用 operator 的 `catalog:import` 用户权限调用 `/api/v1/admin/ingestion`。本地自动测试也验证独立的 internal ingestion 路径。云端人工 CLI 经 HTTPS ALB 管理入口，不开放 `/internal/v1`，也不假定公网 CLI 能解析 Service Connect 地址。不得通过直接写 SQL 绕过 schema validation、review 和 publication rules。可由同一负责人兼任 reviewer / publisher，但各动作仍需独立 audit record。

从空库启动时，顺序为 Flyway schema → operator 登记 Source policy / Restaurant / Branch → 创建 SourceRecord → 创建 batch → 上传 records chunks → complete → review → publish。Synthetic fixture bootstrap 同样调用这些 application services，固定身份由本地或测试 profile 提供；真实来源权限必须由人工核实，不能从输入 CSV 中直接信任。

上传文件时由 Java 签发受限 presigned URL，限制目的 bucket / key 和有效期；完成后验证对象归属、大小、类型、hash 和来源 policy，再登记 SourceRecord。无文件的手工来源也要记录真实来源与 observation，不要求为满足字段而伪造对象。

### 8.2 Batch 示例

```json
{
  "source_id": "src_merchant_demo",
  "scope": {
    "branch_id": "branch_demo",
    "channel": "MERCHANT_DIRECT",
    "fulfillment_type": "PICKUP"
  },
  "schema_version": "1",
  "idempotency_key": "merchant-demo-menu-20261028-observation-01",
  "observed_at": "2026-10-28T01:00:00Z",
  "records": [
    {
      "external_record_id": "menu-item-001-regular",
      "name_original": "示例午餐",
      "variant_key": "REGULAR",
      "base_price": "18.50",
      "currency": "AUD",
      "tax_status": "INCLUSIVE",
      "option_groups": [],
      "availability": "AVAILABLE",
      "availability_evidence_type": "LISTED_ON_MENU",
      "source_record_id": "sr_demo_01"
    }
  ]
}
```

该结构是便于阅读的 batch envelope；实际 HTTP 将登记 metadata 和 records 分别发送到对应 endpoint，随后调用 complete 固定 manifest。示例为 synthetic，不是已收集的商家事实。创建接口返回 `batch_id`，records 接口返回逐项 validation result；未经审核不会直接进入推荐。示例中的 SourceRecord 应由 Java 在来源登记 / 上传验证后生成；CLI 只能引用已存在且属于本批来源的记录。

### 8.3 原子性、重试与并发

1. 每个首轮发布批次只覆盖一个 Branch + channel + fulfillment scope；多 scope 导入拆成独立批次并报告每批结果。
2. API 幂等唯一范围为 caller + operation + resource scope + idempotency key；创建 batch 的 scope 为来源和门店渠道，records / complete / review / publish 的 scope 为 batch ID。同一 scope 同 key 同 payload 返回已有结果，同 key 不同 payload 返回 `409 IDEMPOTENCY_CONFLICT`。不同 records chunk 必须使用独立 key 与 hash；complete 核对 expected record count 和完整 manifest hash，缺 chunk 时不得 review。
3. review 与 publish 带 `expected_revision` 和 payload hash；并发改动或已发布版本变化返回 `409 VERSION_CONFLICT`，重新读取与审核。REVIEWED 后改内容先退回 DRAFT，旧 review 失效，重新 complete；PUBLISHED 的内容不可追加，使用新 batch。
4. 发布 transaction 再次校验来源权限、审核 hash、外键、价格、freshness 和当前 revision；写入 immutable snapshot、切换 current pointer、增加 catalog revision、写 audit，一并 commit；启用 outbox 后事件也在同一 transaction 写入。
5. schema validation、解析、模型调用与远程下载不在该 transaction 内执行；任一关键校验失败时整批不发布。
6. 来源权限撤回、影响硬约束的门店状态变更同样增加 catalog revision；这些管理写入与最终推荐使用相同 concurrency control。首轮通过从 MySQL 获取 current revision、版本化 cache key 和旧 key TTL 保证读取正确性，不依赖外部 queue。若需要主动删除或后续通知，outbox dispatcher 在 commit 后按 event ID 幂等处理，失败重试并告警；不把 MySQL commit 与 Redis 操作误当作同一原子 transaction。
7. 查询 cache key 包含 catalog revision；未能确认 current revision 时从 MySQL 查询或返回受控错误。候选有效性还必须按当前时间检查 freshness / source expiry；cache TTL 不超过包含记录的最早有效期。最终推荐始终用 MySQL 核对当前可用版本，不依赖 cache freshness。
8. 模型返回后，Java 在短 transaction 内锁定所涉 current pointers / Source 状态行或采用等价原子 CAS，再同时核对 MenuSnapshot、MenuItemRevision、OfferVersion、所选 options、来源用途与有效期；校验通过和 Recommendation completion 在同一原子边界保存。发布、来源撤回与最终校验使用一致锁顺序。相关版本或 hard facts 变化时返回 `DATA_CHANGED`，不能静默用旧价通过预算，也不自动重调付费 LLM。结果只代表 final commit 时的有效事实，后续读取仍检查撤回与过期。

后续独立 worker 放在 `services/ingestion/`。将来启用 SQS 时按 at-least-once delivery 设计 worker：消息只携带 batch ID / object reference，重复消费不重复发布，超过重试阈值进入 DLQ，重放仍经过同一幂等边界。

## 9. 推荐证据与数据访问

Java 向 AI 只发送通过权限、状态和硬条件筛选的候选，包含 `menu_item_id`、`menu_item_revision_id`、`offer_id`、`offer_version_id`、`menu_snapshot_id`、可展示的 `source_ids`、确定规格与可选 options。仅发送 `llm_processing_allowed=ALLOWED` 的字段；供应商 access token、授权文件和个人资料不进入候选。

模型输出至少包含上述 identity / version 引用、quantity、selected options、reason、source_ids 和 uncertainties。Java 校验这些 ID 属于原候选范围，并以当前业务版本再次检查价格、禁忌和授权。模型说明中的价格若与业务结果不一致，应移除或用业务事实模板重写，不自动再次调用付费模型，不能连同错误金额展示。

Recommendation 保存请求预算与 scope、偏好快照、候选版本、最终选择、来源、实际 provider / model、Prompt / Persona / Skill versions 及完成状态。内容保留期限由用户数据 policy 与来源 policy 共同限制。

## 10. 首轮 fixtures 与验收

| Case | 必须观察到的结果 |
| --- | --- |
| 菜品 18.50 + 必选加料 2.00，预算 20.00 | 不通过预算，即使展示基础价低于预算 |
| 菜品小计 19.00，配送费 5.00，service fee 2.00，预算 20.00 | 预算通过；页面另列费用且不显示总价保证 |
| 同菜在自取 / 两平台分别为 18.00 / 20.00 / 21.00 | 保留独立 Offer，不覆盖彼此价格 |
| 同名不同门店或份量 | 不自动 merge；映射不明确时进入审核 |
| 菜品本体不含乳，但已选或必选酱料含乳 / 未知 | 按最终配置的证据排除 strict 无乳候选 |
| 价格未变，但 MenuItemRevision 的 hard fact 改变 | 最终校验返回 DATA_CHANGED，不只比较 OfferVersion |
| 价格、必选规格、含税口径或 hard allergen fact 未知 | 排除严格候选，返回可理解原因 |
| 来源只允许展示，不允许 external LLM processing | 受限字段不进入 prompt / tracing；不得绕过权限 |
| 原始资料不允许保存 | 不写 raw、持久化日志、cache、队列或 backup |
| 审核后 payload 改变 / 重复导入 / 并发发布 | 分别要求重审、复用幂等结果、返回版本冲突 |
| publish commit 成功但 Redis 不可用 | 数据保持已发布；版本化 key 防止旧结果命中；启用 outbox 时可重试；最终校验不读过期 cache |
| 推荐生成期间涨价、撤回或来源过期 | Java 阻止旧版本成为最终结果 |
| 菜单过期且 scheduler 尚未运行 | 查询时仍排除过期候选 |
| 备份恢复与来源到期删除 | 恢复后重算有效性；不重新公开已撤销或到期内容 |

验收交付包括可再分发的 seed fixture、validation report、审核 / 发布记录、上述案例的实际执行结果以及可重复运行的 import 路径。本文仅定义这些要求，未执行测试或验证生产数据。
