# Large-Lunch-Model：AWS 要求

- 版本：v0.2；更新日期：2026-11-06（Australia/Sydney）。
- 状态：待实现的部署要求与默认方案；尚未创建 AWS 资源、验证账号权限或测得运行费用。
- 上位基线：[技术栈与架构要求](../tech-stack-requirements.md)。
- 相关文件：[User Stories 与验收](sprint-1-mvp-and-user-stories.md)、[数据要求](data-requirements.md)、[本地开发要求](local-development.md)。
- 已确认：保留 ECS Fargate；本次增加首轮 authentication、abuse protection 与 distributed admission 要求，不切换到 Vercel + EC2。
- 应用与业务数据默认 Region：`ap-southeast-2`（Sydney）。Cloudflare 与 CloudFront 是 global services；外部 model inference 的处理位置取决于 provider，不能由应用 Region 推断。

## 1. 部署目标与阶段

首轮目标是在受限访问的 staging 上运行一条完整推荐路径：注册登录且邮箱已验证 → Turnstile → Java 原子 admission → 有来源的 seed data 查询与过滤 → 一个 model provider → Java 校验 → 页面展示。匿名用户仍可浏览、搜索和打开外部链接；任何新付费 AI generation 均不得通过匿名 API 或公开概览 cache miss 绕过上述入口。

本地开发不依赖 AWS 账号。只有进入云端部署时，才需要确认账号、权限、域名、provider credentials 和费用上限。

| 阶段 | 必须具备 | 按实际功能加入 |
| --- | --- | --- |
| 首轮 staging | VPC、ALB、ECR、三个 Fargate services、RDS MySQL、S3、Cognito、Cognito User Pool 的 AWS WAF、Cloudflare proxy / WAF / rate rules / Turnstile、Secrets Manager、CloudWatch、CI OIDC、Service Connect TLS、受限访问、一次性 migration 路径、共享 admission ledger | CloudFront 支持独立素材分发；Redis 仅在 cache / 辅助 prefilter 确有用途时部署 |
| 受控 Beta / production | 独立环境与 credentials、恢复演练、容量与成本上限、发布及故障责任、稳定的身份与权限、必要的 HA | Multi-AZ RDS、每 AZ NAT、每服务多副本、ElastiCache HA，按可用性目标配置 |
| 自动 ingestion | 独立 worker、SQS、DLQ、幂等与重试、任务可观测性 | EventBridge Scheduler 定时更新；仅在需要定时获取来源时启用 |

首轮手工导入由 operator 使用 Cognito access token 与 `catalog:import` 权限，经 Cloudflare → ALB 的 `/api/v1/admin/ingestion/*` 调用 Java；来源登记、review、publish 使用各自的管理 scopes。内部 ingestion endpoint 只留给 private worker。同一 application service 统一校验，不开放公网 `/internal/v1`。Standalone task 不支持 Service Connect，不能假定一次性 CLI 自动解析其 alias；migration task 直接连接 RDS DNS，不依赖这些 alias。

用户预算使用 `MENU_SUBTOTAL`：所选菜品、规格、必选加料、实际选择的可选加料和数量计入；delivery fee 与平台 service fee 另列。AWS 费用预算是团队基础设施开支，与用户的餐饮预算相互独立。

## 2. 环境与资源隔离

- `local`、`staging`、`production` 使用不同配置、数据库、S3 bucket、Secrets 和 model usage 配额。
- 首轮只有 staging；production 资源在确有上线需求后创建。production 优先使用独立 AWS account；若暂用同一 account，仍须独立 VPC、IAM roles、state 和数据资源。
- staging 不复制 production 私人对话或用户信息；演示使用可分发 fixture 或获得授权的数据。
- 资源命名采用 `llm-<environment>-<component>`，其中 `llm` 仅为项目缩写。
- 至少设置 `Project=Large-Lunch-Model`、`Environment`、`Owner`、`ManagedBy=Terraform` cost allocation tags。
- runtime、container architecture、engine versions、区域可用性和 service quotas 在首次部署时记录，使用经过验证的明确版本。

## 3. 网络与流量路径

```mermaid
flowchart TD
    B["Browser"] -->|"HTTPS"| EDGE["Cloudflare proxy / WAF / rate rules"]
    EDGE -->|"Full strict TLS / origin verification"| ALB["Public ALB :443"]
    B -->|"PKCE"| AUTH["Cognito managed login + AWS WAF"]
    B --> TS["Turnstile widget"]
    JAVA -->|"Siteverify"| VERIFY["Cloudflare Turnstile API"]
    ALB -->|"页面"| WEB["Private Next.js :3000"]
    ALB -->|"/api/v1/*"| JAVA["Private Java :8080"]
    WEB -->|"SSR 查询 / Service Connect"| JAVA
    JAVA -->|"内部认证 / Service Connect"| AI["Private FastAPI :8000"]
    AI -->|"受控只读 tools"| JAVA
    JAVA --> DB[("Private RDS MySQL")]
    JAVA --> CACHE[("Optional Redis")]
    AI --> NAT["NAT Gateway"]
    NAT --> MODEL["External model API"]
    JAVA --> RAW["Private S3 raw"]
    B --> CF["CloudFront assets"]
    CF -->|"OAC"| ASSET["Private S3 assets"]
```

### 3.1 Subnets 与出站

- VPC 至少规划两个 AZ，每个 AZ 包含 public subnet、application private subnet；数据库使用覆盖两个 AZ 的 DB subnet group。
- ALB 位于 public subnets。Fargate services、RDS 和 Redis 位于 private subnets；Fargate `assignPublicIp=DISABLED`，RDS `publicly_accessible=false`。
- 首轮 staging 默认一台 NAT Gateway 提供 internet egress；这是控制起步成本的选择，存在 NAT 所在 AZ 故障影响及跨 AZ 流量成本。
- 进入有 HA 目标的 production 时，评估每个 application AZ 独立 NAT，并让私有子网使用同 AZ 出站路线。
- Fargate 拉取 image、读取 Secrets、发送 logs、访问 model provider 所需网络路径必须逐一验证。只有 private subnet 并不代表这些操作自动可达。
- S3 gateway endpoint 可用于 S3 私有访问；ECR、Secrets Manager、CloudWatch Logs 等 interface endpoints 按费用和网络需求决定。AWS service endpoints 不提供到任意外部 model API 的 internet egress。
- 首轮采用 IPv4 路线；IPv6 / dual-stack 优化需单独验证外部 provider、image registry 和安全规则的兼容性。

此网络方案依据 [ECS internet connectivity](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/networking-outbound.html) 与 [Fargate networking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html)。

### 3.2 ALB routes 与 Security Groups

| 入口 / 路径 | 要求 |
| --- | --- |
| ALB `443` | 只接受 Cloudflare origin 流量；转发规则必须同时匹配允许的 Host 和 per-environment origin secret header，default fixed `403` |
| ALB `80` | 首轮关闭；HTTP → HTTPS redirect 在 Cloudflare edge 完成 |
| `/api/v1/*` | 路由至 Java；所有写操作与用户数据读取执行相应授权 |
| `/api/v1/admin/*` | Java 校验 admin / reviewer 权限；普通登录用户不能导入、审核或发布数据 |
| `/internal`、`/internal/*`、`/actuator`、`/actuator/*` | 在 public listener 使用高优先级 fixed `404`，不能落入默认 Next.js 路由或被转发至 Java |
| 其他页面路径 | 路由至 Next.js；不设置可透传任意内部 URL 的 proxy |

- ALB Security Group 的 `443` 只允许当前发布的 Cloudflare origin IP ranges；初始 IPv4 origin 只开放适用 IPv4 ranges，若启用 dual-stack 则同时维护 IPv6 ranges。不能把 reviewer 的真实 IP 加在 ALB Security Group；staging reviewer allowlist 在 Cloudflare edge 判断真实 client IP。
- Web Security Group 只允许 ALB Security Group 访问 `3000`；Java 只允许 ALB、Web、AI 和已授权 ingestion service 的必要流量。
- AI 的 `8000` 只接受 Java Security Group；不能创建通向 FastAPI 的 public ALB target route。
- RDS 的 `3306` 只接受 Java runtime 和 migration task Security Groups。AI、Web 与 ingestion worker 不持有业务数据库 credentials。
- Redis 的 `6379` 只允许实际使用它的服务；若 Python 后续确需 AI 临时缓存，使用独立 namespace、credentials 和最小权限。
- Service Connect 的命名、port mappings 和 proxy ingress ports 显式配置；Security Groups 必须匹配最终端口，不能仅凭容器应用端口推断。
- ALB path rules 是入口防护，Java 仍须拒绝 browser credentials 调用内部接口；路径变体、大小写和编码变体由 router 统一规范并做负向测试。

Service Connect 提供服务发现与连接管理，不代替应用 authorization。按 [AWS Service Connect 文档](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect.html) 为 proxy 预留 task CPU / memory。

### 3.3 DNS 与 TLS

- 应用域名的 DNS zone 默认由 Cloudflare 管理并启用 proxy；注册商不必迁移，Route 53 可保留其他独立 zones。ALB 使用位于 `ap-southeast-2` 的 ACM certificate，DNS validation records 保持 DNS-only。Cloudflare → ALB 使用 Full (strict)，校验 origin certificate 的有效期、信任与 hostname；禁止 Flexible 或关闭 certificate validation。[Full (strict) 文档](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/)
- CloudFront 使用自定义域名时，其 viewer certificate 必须位于 `us-east-1`。这不改变 RDS 或 raw data 的 Region。[CloudFront certificate 要求](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html)
- 第一轮 CloudFront 仅分发已授权素材；页面与 SSE API 经 Cloudflare → ALB。Cloudflare 对 `/api/v1/*`、authorization 请求和私人页面设 cache bypass；SSE 响应 `Cache-Control: no-store`，不启用缓存、HTML transformation 或会 buffer stream 的优化。独立 assets 域名与 API 路径不得混用 cache policy。
- 云端 service-to-service 默认启用 Service Connect managed TLS，并使用 AWS Private CA 的 short-lived certificates；CA、certificate rotation、Secrets 和适用的 KMS 费用纳入预算，不能把 managed TLS 当作免费附带能力。
- 首轮明确采用 `awsvpc` + Service Connect managed TLS：Web / Java 的 ALB target groups 使用 HTTPS，target port 与 task 的 `containerPort` 一致，分别为 3000 / 8080；health checks 同样 HTTPS，使用 `/healthz` 与 `/actuator/health`。不设置 `ingressPortOverride`。流量由 Service Connect proxy 终止 TLS，再在 task 内经 localhost HTTP 到应用；不要求 Next.js / Java 自己配置 HTTPS certificate。ALB 443 listener 选择支持 TLS 1.3 的 security policy，并按官方组合验证 targets healthy。[Service Connect TLS 文档](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect-tls.html)
- Web、Java、AI 作为 Service Connect-enabled services 加入同一 namespace，应用使用配置的 service alias；proxies 之间启用 TLS。ALB 本身不需要成为 Service Connect client。Security Groups 放行实际 container/proxy port，禁止通过额外端口绕过 proxy。[Service Connect components](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect-concepts-deploy.html)
- TLS provisioning 使用独立 ECS infrastructure role，按官方 policy 允许 Private CA、Secrets 与相应 KMS 操作；该 role 不替代应用 task / execution roles。Private CA 标记 `AmazonECSManaged=true`，Terraform dependency 确保服务排空后才销毁 CA、相关 role 或 KMS key；没有 CA 和该 role 时不能把部署标为已完成 TLS。
- 若初始化时因成本改用应用 TLS，需要先记录替代方案的证书签发、信任和轮换方式；不得静默退回以明文传送内部 bearer token。TLS 与 service identity / scope 校验分别实施。

### 3.4 Cloudflare edge 与 origin 防绕过

- 记录 Cloudflare account / zone owner、plan、规则配额、域名和 key owner。启用实际 plan 可用的 managed WAF rules、custom rules、DDoS protection 和 rate limiting；按路径区分公开目录、AI 请求、私有读取与 admin，不用一个 IP 阈值替代用户额度。rate limiting 的规则、计数粒度和 capabilities 受 plan 限制，edge counters 不是精确 billing / quota ledger。[Cloudflare rate limiting](https://developers.cloudflare.com/waf/rate-limiting-rules/)
- 除 Security Group allowlist 外，Cloudflare Request Header Transform Rule 必须覆盖客户端传入值并设置高熵 `X-LLM-Origin-Verify`。ALB 对预期 Host + 当前 secret 的组合放行，其余 default deny；仅允许 Cloudflare IP 并不能限定为本项目的 zone。[Origin protection](https://developers.cloudflare.com/fundamentals/security/protect-your-origin-server/)、[Request Header Transform Rules](https://developers.cloudflare.com/rules/transform/request-header-modification/)
- Origin secret 按环境隔离，仅供 Cloudflare rule 与 ALB listener condition 使用；不传给浏览器、不回显、不进入 request logs。若 IaC provider 会把值写入 state，必须把 state / plan 作为含 secret 的敏感资料加密并限制访问，不能声称标为 `sensitive` 就不会保存。轮换采用短期双值接收 → 更新 edge → 验证 → 删除旧值，记录 owner 与截止时间。
- 每次修改 Cloudflare IP ranges、Host、Transform Rule 或 ALB listener 后，验证正常请求能通、直连 ALB DNS / origin IP 被阻断、伪造 header 无法绕过。禁止为 smoke test 长期开启 `0.0.0.0/0`、DNS-only API alias 或其他 public listener。
- Java 只有在已建立上述可信 origin channel 时才采用 `CF-Connecting-IP` 作为 client IP；不能相信任意 `X-Forwarded-For`。IP 只用于粗粒度防滥用；按用户的业务额度使用服务端身份，避免共享校园 / 家庭 NAT 导致所有人共用一个 AI quota。
- 对 API / SSE 使用 block 或 rate-limit responses；不要在 JSON / SSE 中注入 HTML Managed Challenge。页面嵌入 Turnstile，前端对 edge 拒绝、应用 `429`、`503` 分别显示可理解状态，不自动重放付费请求。
- 外层 Cloudflare rule 不保护 Cognito 自身 hosted domain；身份入口的独立保护见 6.1。Cloudflare 也不代替 application authentication、authorization、input validation 或来源事实校验。

## 4. Fargate services 与应用运行

| Service | Container port | 权限与运行约束 |
| --- | --- | --- |
| Next.js | `3000` | 页面、公开目录 SSR；无 DB 或 model key。私人数据首轮由浏览器内存 access token 执行 client fetch，不假定 SSR 获得 token |
| Java business API | `8080` | 唯一业务数据库 writer；验证用户和 service identity；执行过滤、审核、发布与最终推荐校验 |
| FastAPI AI application | `8000` | 调用 model API；只读 tools 通过 Java；不拥有数据库或 ingestion publish 权限 |
| Ingestion worker（后续） | 无 public port | 读取获准 raw objects，提取与清洗，通过限定的内部 ingestion API 提交 draft |
| Migration task | 无 public port | 一次性任务；专用 DDL credentials；成功结束后才继续 release |

- 每个应用具有独立 image、task definition、service、log group、execution role 和 task role。
- 首轮 staging 日常可每服务一副本，不宣称 HA；首次 release 前将 Java 与 AI 各临时扩至至少两副本，以少量 live challenge / provider smoke 验证 routing、shared ledger 与 rolling deployment，结束后可缩回。完整 50 / 10 / 20 并发案例在隔离的 local / CI 中执行，见 5.4 节。进入 production 前根据 availability 目标决定副本数、AZ 分散与 rolling deployment 余量。
- 初始 CPU / memory 由实际 image 启动测试确定，Java heap、metaspace、native memory 与 Service Connect proxy 必须共同纳入 memory budget。
- Fargate CPU / memory 使用合法组合；设置资源上限、非 root user、必要的 writable `/tmp` 和 graceful shutdown。
- 使用与 image 匹配的 `runtimePlatform.cpuArchitecture`；在 Apple Silicon 构建 image 时验证目标 architecture，不能默认本地 image 可直接运行在云端。
- ECR image 使用 commit tag 加 immutable digest；生产 release 按 digest 部署，禁止依赖可变 `latest`。
- 开启 image scanning、配置 lifecycle policy；至少保留当前 release 与已验证的 rollback image，不能在清理中删除仍被运行任务引用的 image。

### 4.1 Health checks 与 streaming

- Next.js 提供轻量 `/healthz`；Java 使用内部 `/actuator/health` 或专用 readiness group；FastAPI 提供 `/healthz`。
- ALB target health check 直接请求 target，不要求 public listener 暴露 health path；响应不包含 credentials、连接字符串或内部拓扑。
- liveness 不调用外部 LLM；provider 短暂故障不能触发全部服务不断重启。readiness 与 dependency 状态按业务需要区分。
- SSE 建议初始 heartbeat 为 15 秒、ALB idle timeout 为 120 秒；provider connect / LLM operation / Java request deadline 初值分别为 5 / 60 / 90 秒。Service Connect HTTP endpoints 显式配置 `perRequestTimeoutSeconds=120`、`idleTimeoutSeconds=120`，不能沿用可能在完整 stream 结束前触发的默认 15 秒 per-request timeout；较短 application deadline 优先生效。以上均为待 smoke 验证的项目初值。[Service Connect timeout](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect-concepts-deploy.html)
- Application connection timeout、proxy idle timeout、ALB idle timeout 和 shutdown drain 共同验证；ALB 的 HTTP/2 PING 不会重置 idle timeout。[ALB timeout 文档](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html)
- 不 buffer 整段 SSE；每个 event flush，禁用会延迟小数据块的压缩或中间 buffering。通过真实 Cloudflare → ALB 路径验证首个 event、15 秒 heartbeat、idle / total deadline；把当前 plan 的 proxy timeout 纳入记录。浏览器断开后尽力取消下游调用并记录最终状态；断开不证明 provider 已停止，不能立刻释放仍可能运行的 capacity reservation。
- rolling deploy 的 deregistration delay 与应用 shutdown timeout 必须让现有请求有明确结束结果；超过期限按中断处理，不宣称支持断线续传。
- CPU autoscaling 之外还监测 active streams、连接数和 provider concurrency；扩容副本不能绕过团队的总 token / request budget。

## 5. 数据存储与访问

### 5.1 RDS for MySQL

- 选用与本地和 CI 一致、仍受支持的 MySQL major version；实际 minor version 在初始化记录，先验证 MyBatis、Flyway 和 driver 组合。
- 使用 private DB subnet group、TLS database connections、静态加密、独立 runtime user 与 migration user；runtime user 无 DDL / grant 权限。
- Java connection pool 的总容量按最大 task 数计算，并为 migration、运维和数据库内部工作保留余量。
- staging 默认可用 Single-AZ；production 的 Multi-AZ 由 availability 目标决定。Multi-AZ 不替代备份，也不自动提供应用级 rollback。
- 首轮建议 automated backup retention 为 7 天；production 初值为 14 天，具体保留期根据数据要求和费用确定。
- production 开启 deletion protection，Terraform 配置保护持久资源，删除前要求 final snapshot 和明确的数据处置记录。
- storage autoscaling 设置上限，并监控剩余空间；自动扩容不代替成本控制。
- Source / OfferVersion / 推荐记录的保存与删除条件遵循[数据要求](data-requirements.md)；备份保留与来源删除要求一起管理。

RDS 支持指定 retention period 内的 point-in-time recovery；恢复必须实测，不能由“已开启 backup”推断已具备恢复能力。[RDS backup 文档](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.html)

### 5.2 S3 与 CloudFront

至少分离以下三个 bucket；名称是模板，需替换为全局唯一的实际名称。

| Bucket 类别 | 内容 | 访问边界 |
| --- | --- | --- |
| `llm-<env>-raw-<unique>` | 获准保存的原始菜单、提取中间产物 | Private；限定 Java / ingestion roles；不关联公开 CloudFront distribution |
| `llm-<env>-assets-<unique>` | 获准公开展示的图片、Persona 素材 | Bucket 保持 private，通过 CloudFront OAC 读取指定 prefix |
| `llm-tfstate-<unique>` | Terraform state 与 lockfile | 仅 infrastructure roles；与应用数据 bucket 分离 |

- 三类 bucket 均启用 Block Public Access、加密和禁止非 TLS 的 bucket policy；应用不能通过 public ACL 公开 raw object。
- assets 使用 S3 REST origin 与 OAC，并以 distribution ARN 限定 bucket policy；OAC 不适用于 S3 website endpoint。[CloudFront S3 access 文档](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- raw objects 的 retention、versioning 和 lifecycle 以来源许可为依据；删除必须覆盖旧 versions 与中间产物，不能只创建 delete marker。
- assets 使用带版本或 content hash 的 object key；删除或撤回授权时处理 CDN cache，不能只删数据库引用。
- presigned upload URL 由 Java 在鉴权后签发，限制 object key、有效期、文件类型及大小策略；上传完成后验证实际 object 并登记 metadata。
- 原始文件默认 private；审核人员通过短期授权下载。`source_url` 不等于公开 raw object URL，也不保证第三方内容可再分发。
- S3 Event notifications 若后续启用，同样按可能重复或乱序的事件处理，不能触发未经审核的自动发布。

### 5.3 Redis 与异步任务

- Redis 是可重建 cache / rate prefilter / temporary state；目录事实、推荐历史和 authoritative admission ledger 始终在 MySQL。
- 无论是否启用 Redis，都由同一 MySQL transaction 原子预留 user / global / provider concurrency slots 与 request / token budget；进程内限流只做附加防护，禁止 Redis 与 MySQL 各维护一套可独立放行的 quota。
- 模型调用前完成短 transaction 的 quota reservation，外部 HTTP 调用期间不持有 DB lock；结束后结算 usage。进程中断或 usage 不明时保守保留 reservation，由 reconciliation 处理，避免重复释放额度。
- 发布菜单时在 MySQL transaction 内更新 `catalog_revision`；查询先取得 authoritative revision，再使用带 revision 的 cache key，旧 key 依 TTL 清理。首轮不为 cache invalidation 强制引入 SQS 或 outbox。
- ElastiCache 选型按 engine compatibility、最低运行成本、HA 和 workload 比较；不预设 Serverless 一定更便宜。
- 使用 encryption in transit、适用的 ACL / authentication 和 private network；Redis 失败时目录查询可回源，AI 仍须通过同一 MySQL admission。MySQL admission 不可用则拒绝新付费请求，不降级到本地 counter。
- SQS 消息只包含 task ID、source ID、object reference 和 schema version，不能放完整菜单、credentials 或用户私人对话。
- Worker 使用 visibility timeout、bounded retries、DLQ 与 idempotency key；数据库任务状态与消息重复处理结果可追踪。
- EventBridge Scheduler 仅负责触发；数据能否发布由 Java 的校验与审核规则决定。

### 5.4 首轮 admission、并发与额度

以下是项目的可配置初值，不是压测结果、provider capacity 或 SLA；以总体 framework 的同名配置为 contract，变更须记录 `admission_config_version` 与生效时间。所有环境使用自己的 ledger 与 provider credentials。

| 配置 | 初值 | 口径 |
| --- | --- | --- |
| `AI_MAX_ACTIVE_PER_USER` | `1` | 同一用户跨 tab、session 和 replica 的 active attempts |
| `AI_MAX_ACTIVE_GLOBAL` | `5` | 同一环境的全部 AI operations 共用 |
| `AI_MAX_ACTIVE_PER_PROVIDER` | `5` | 同一环境、当前 provider 的合计；扩容不增加额度 |
| `AI_REQUESTS_PER_USER_PER_MINUTE` | `5` | rolling 60 秒的新 turn 尝试，失败验证也计数；同 key 查状态不算新 turn |
| `AI_REQUESTS_PER_USER_PER_DAY` | `20` | UTC 日已开始 / 可能已开始的 provider attempts；调用前先 reservation |
| `AI_TOKENS_GLOBAL_PER_DAY` | `100000` | UTC 日 input + output tokens；先预留上界再结算 |
| `AI_MAX_INPUT_TOKENS` / `AI_MAX_OUTPUT_TOKENS` | `6000` / `1000` | input 包括 system、history、候选和 tool schema；output 上限与 reservation 包含适用 reasoning tokens，不可界定收费上界的 model 不启用 |
| `AI_MAX_CALLS_PER_TURN` | `1` | 首轮一个 provider call，不自动付费 repair / summarization / retry |
| `AI_MAX_MESSAGE_CHARS` / `AI_MAX_REQUEST_BYTES` | `2000` / `32768` | 单条输入按 Unicode code points 计数；读取 body 时即限制 bytes |
| `AI_QUEUE_ENABLED` | `false` | 容量满直接 backpressure，不建无上限等待队列 |

- Admission、idempotency claim 和 conversation version 使用可跨 replica 的数据库约束；按 framework 第 5.5 节依次限制 body / attempt、验证身份与 owner、创建唯一 pending request、检查 user attempt rate、可信邮箱 / Siteverify、候选与 Prompt token 上限，然后原子 reservation。网络调用不在持锁 transaction 中执行，冲突有 bounded retry；失败不得开始 provider call。
- Java 签发 request-scoped context；Python 必须调用 `POST /internal/v1/ai-requests/{id}/claim` 原子领取唯一 dispatch 资格，才允许调用 provider。usage / 终态通过 `POST /internal/v1/ai-requests/{id}/usage` 交给 Java 幂等结算，Python 不直接写 DB。重复内部 HTTP delivery 不产生第二次 claim；领取后是否已到达 provider 不明时保留 reservation，不自动 dispatch。
- 无候选不占用 daily AI quota；已预留但能够确认 provider 未开始的失败释放 request / token reservation。已开始或无法确认是否开始的失败、cancel、timeout、DATA_CHANGED 均计入当日 request 次数。usage 未知则保守持有 token reservation，结算采用条件更新保证幂等。跨 UTC 零点的请求归属 reservation 创建时的 bucket，active slots 不因换日被重置。
- 每个 active attempt 使用 lease 和 fencing token；初始 lease 90 秒、每 15 秒 renew。lease 到期只触发 reconciliation，不是立即新增 capacity 的证据；provider 状态未知时进入 `RECONCILIATION_REQUIRED` 并保留 reconciliation hold，必要时暂停对应 provider 的新增 admission。延迟返回的旧 worker 不能凭旧 fencing token 覆盖终态或释放新 reservation。
- 60 秒 LLM operation deadline、90 秒 Java deadline 保持不变。应用可以保证不发起超过 admission 上限的新请求，但不能把客户端 timeout 当作外部 provider 已终止计算的证明；未知调用须按 adapter 能力核对，超时无法确认时保持保守 hold 并告警。
- `429` 区分 `USER_RATE_LIMITED`、`USER_CONCURRENCY_LIMITED`、`USER_DAILY_QUOTA_EXCEEDED`；`503` 区分 `AI_CAPACITY_EXHAUSTED` 与 `AI_TEMPORARILY_UNAVAILABLE`。返回适用的 `Retry-After` / `reset_at`，不承诺届时必有空位。SSE 建立前用 HTTP JSON，建立后用 `error` + `done`；不用 `200` 伪装 admission 拒绝。
- 公开 cached AI overview 只读；cache miss 不能隐式触发模型。新的 overview generation 与 chat 共用额度和 limits。真实 provider smoke 使用已验证的专用测试账号，经同一 live Turnstile / admission 路径执行，并计入同一环境 / provider 上限；首轮不另开匿名或不受额度约束的 smoke endpoint。

完整 50 用户、单用户 10 tabs、20 个重复 key 的负载与故障注入测试，在隔离 local / CI 的共享 MySQL、两 Java / 两 AI replicas 上使用显式 mock auth / challenge 与 barrier mock provider 执行。Fargate staging 只以少量 live Turnstile / provider smoke 验证双副本 routing、shared ledger 和 rolling；不能为自动化 load test 关闭公开 staging 的 gate，也不能把本地测试报告冒充云端容量证明。

## 6. 身份、IAM 与 Secrets

| 身份 | 最小职责 |
| --- | --- |
| 人类开发 / 运维 | IAM Identity Center 或受控 federation，MFA；不日常使用 root user |
| CI build role | 推送限定 ECR repositories；无 production DB 或 raw data 读取权 |
| CI deploy role | 更新限定 ECS services、运行 migration task；`iam:PassRole` 只允许明确的 task / execution roles |
| CI infrastructure role | 管理该环境的 Terraform resources；与日常应用 deploy role 分离 |
| ECS infrastructure role for Service Connect TLS | ECS 管理该环境的 Private CA certificates、相关 Secrets / KMS；不能授给 application |
| Task execution role | ECS 拉取 image、写 container logs、读取 task 启动注入所需的具体 Secrets |
| Task role | 应用运行时通过 AWS SDK 所需的 S3 / SQS 等权限；按服务独立。Java 另有仅针对该环境 User Pool ARN 的 `cognito-idp:AdminGetUser`，无全局 User Pool 管理权限 |

Task role 与 execution role 的职责不同；不把宽泛 administrator policy 挂给应用。[ECS task role 文档](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)

- 首轮远程 staging 用户和 operator 使用 Cognito；注册、邮箱确认、登录、退出和 password reset 属于首轮功能。浏览器 Authorization Code + PKCE，access token 在内存，私人内容通过 client fetch；CLI 使用单独的 operator 登录，不存长期用户 token。Java 检查 JWT signature、issuer、`token_use=access`、expiry、`client_id` 与 scope / group 的适用规则，并以可信 `sub` 映射应用 user ID。
- AI generation 必须同时满足已登录、邮箱已验证、application account active；每个新付费 turn 由 Java 使用限定 IAM 权限调用 `AdminGetUser` 核实 Cognito `Enabled` 与 `email_verified`。不能假定 access token 一定含 `email_verified`，也不能接收浏览器自报；identity 检查失败或不可用时拒绝新付费请求。该调用量与适用 Cognito billing / MAU 规则须计入 worksheet。
- 应用 MySQL account 状态与 `AuthSession` 每次鉴权检查；启用 Cognito token revocation，以可信 `origin_jti` 关联 token family。`POST /api/v1/auth/logout` 先持久化当前 family 的应用撤销，再由浏览器完成 token revoke / Cognito logout 与本地清理。仅当前 family 的重复 logout 在 JWT 其他验证仍有效时允许幂等成功，不授予其他私有 API 访问。Cognito revoke 或退出并不等于离线 JWT 验签会立即失效；不依赖 JWT 到期实现应用 logout。
- Java → AI 使用 `BUSINESS_TO_AI_TOKEN`；AI → Java 使用 `AI_TO_BUSINESS_TOKEN`，仅有受 request context 约束的 read tools、dispatch claim 与 usage 回报 scopes；ingestion → Java draft import 使用 `INGESTION_TO_BUSINESS_TOKEN`。三个 secrets 分别分配 caller identity 与 scopes，不能通用。claim / usage 不赋予目录写入或 admin 权限。
- 用户范围与请求上下文以 Java 验证后的数据为准；需要传递签名上下文时由 Java 产生并校验 audience、expiry 与 request 绑定，浏览器不能自选内部 user ID 或 scope。
- ingestion identity 不得拥有 publish 权限；AI identity 不得拥有 ingestion write 或 admin 权限。
- model API keys 仅提供给 AI task；数据库 runtime credentials 仅提供给 Java；migration credentials 仅提供给一次性 migration task。
- Secrets Manager 按环境和服务命名，application secrets 的 Terraform 通常只管理 metadata 与 ARN，不主动把 secret value 放入 `.tfvars`、build args、日志或 frontend variables。Origin header 的 listener / rule 配置可能包含在 Terraform state / plan 中，按 3.4 节的敏感 state 规则保护；不得用“只存 ARN”误述这项 provider 行为。
- 若使用 ECS environment injection，rotation 后必须发布新 tasks；已运行容器不会自动刷新注入值。[ECS secret injection 文档](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/secrets-envvar-secrets-manager.html)
- 登录 token、Turnstile token、origin secret、presigned URL、完整 prompts、私人 preferences 和支付信息不进入常规日志；必须记录时执行最小化、redaction 与保留期限制。

### 6.1 Cognito 入口与 Turnstile

- Cognito User Pool 关联同 Region 的 regional AWS WAF web ACL，配置适用的 managed rules / rate-based rules，结合 email confirmation、password policy 与 Cognito throttling。managed login 和 public User Pool API 都应进入验证范围；不能声称应用页面的 Turnstile 能保护直接访问 Cognito hosted URL 的注册请求。[Cognito 与 AWS WAF](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-waf.html)
- Cognito 不向 WAF 转发 managed login request body；不要设计依赖 email / password 明文匹配的规则。避免给全部 identity requests 强制 CAPTCHA 导致 managed login flows 失败；验收注册、确认、登录、找回密码和合法 shared-IP 用户，记录误拦截处理方式。
- 每个新的付费 AI turn 使用 Turnstile Managed widget；可以静默完成，不要求每次手动点击。Java 调用 Siteverify，chat 仅接受 `success=true`、匹配允许 hostname 和 `action=ai_chat` 的结果；其他按需生成入口启用时定义独立 expected action allowlist，并复用同一 admission service。token 有效期为 300 秒且只能验证一次，过期或 replay 必须刷新；只有前端 widget 不构成防护。[Turnstile server-side validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)
- 幂等重放先匹配已认证 owner、operation、payload hash 与已有 request；不得再验证已用 token 或重新计费。新 key 的并发请求只有一个 verification owner；Siteverify transport retry 复用同一 `idempotency_key`，与业务 Idempotency-Key 分开管理。transport 不确定时 fail closed，不创建第二个模型调用。
- Browser 仅获得 public site key。`TURNSTILE_SECRET_KEY` 只注入 Java task；记录 `TURNSTILE_ALLOWED_HOSTNAMES`、`TURNSTILE_EXPECTED_ACTION=ai_chat` 和环境。Siteverify failure 返回 `BOT_VERIFICATION_FAILED`；不可用返回 `BOT_VERIFICATION_UNAVAILABLE`，保留输入并支持用户重试，不能自动跳过验证。
- local mock / 官方 test keys 只用于 local / CI。远程 staging 与 production 必须使用各自 live keys、hostname allowlist 和真实 Siteverify；启动时拒绝 mock/test bypass。不得用 test key 的成功作为云端防护验收。[Turnstile testing](https://developers.cloudflare.com/turnstile/troubleshooting/testing/)
- Cognito、Turnstile、origin verification 与内部 service token 都分别轮换和授权；任意一层成功都不能代替后续 authorization 或 quota admission。

## 7. Terraform 与 GitHub Actions

### 7.1 Bootstrap 与 state

1. 使用受控人类身份 bootstrap state bucket、GitHub OIDC provider 和环境 roles；bootstrap 必须可重建并记录 owner。
2. state bucket 启用 encryption、versioning、Block Public Access 和访问限制；state 视为敏感资料，不提交 Git。
3. Terraform S3 backend 启用 `use_lockfile = true`；给 lockfile 必要的 Get / Put / Delete 权限，state 本身的权限单独限制。
4. 锁定支持该能力的 Terraform version 与 AWS provider version，提交 `.terraform.lock.hcl`；新项目不新增已 deprecated 的 DynamoDB locking dependency。
5. staging / production 使用独立 backend key 与 IAM boundary；prefer 独立 root configuration，防止一个 workspace 切换误操作环境。
6. `plan` artifact 限定访问与保留期；`apply` 使用审核过的同一 plan，禁止并发修改同一环境的 state。

`use_lockfile` 与 DynamoDB deprecation 以 [Terraform S3 backend 文档](https://developer.hashicorp.com/terraform/language/backend/s3) 为依据；这些 state locks 与 S3 Object Lock 是不同功能。

### 7.2 OIDC 与 release pipeline

- GitHub Actions 使用 OIDC 短期 credentials；workflow 设置所需的 `id-token: write`，不保存长期 AWS access keys。
- IAM trust policy 同时限制 `aud=sts.amazonaws.com` 和精确的 repository / branch 或 GitHub environment `sub`。
- 使用 environment 型 `sub` 时，在 GitHub environment 配置允许部署的 branch / tag；不能假设 JWT 的 `sub` 仍是 branch 型格式。
- production environment 配置保护与 release 权限；不可信 fork PR 的测试 job 不获得部署 roles 或真实 model keys。
- GitHub Actions 第三方 actions 固定可审查的 commit SHA；workflows 最小化 token permissions。

OIDC 条件依据 [GitHub 官方 AWS OIDC 指导](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)。

Release 顺序：

1. lint / type checking、unit / integration / contract tests；常规 CI 使用 mock model provider。
2. 构建并扫描三个 image，推送 ECR，记录 digest、schema migration version 与 AI config versions。
3. 验证待发布代码与现有 schema 的兼容性；必要时完成 snapshot 或恢复点记录。
4. 在 private subnet 启动一次性 migration task，等待明确成功；失败时停止后续 rollout。
5. 更新 ECS services，等待 health checks 和 deployment stability；不在每个 application replica 启动时并行执行 Flyway migration。
6. 导入或选择已审核的 seed data；确认隔离 local / CI 的两 Java / 两 AI mock concurrency 验收通过。在 Fargate 两 Java / 两 AI replicas 下执行少量 live Turnstile / provider smoke、shared ledger / rolling 检查和 origin 防绕过，再完成 Cloudflare / Cognito E2E；不关闭公开 staging 的 authentication 或 human verification。记录 latency、结果、来源、usage 与 admission_config_version。
7. 仅在 production 环境和验收条件已建立后，将相同 image digests promote；production 使用自己的 secrets 和 data，不能复制 staging credentials。首轮 staging pipeline 到第 6 步结束。

Application rollback 使用已验证的上一版 image；数据库采用兼容性 migration 与 roll-forward 为优先。数据库恢复会影响恢复点后的写入，不能把恢复 snapshot 当成无损的应用 rollback。

## 8. 监控、成本与恢复

### 8.1 最小监控

- CloudWatch logs 使用统一 `request_id` / `ingestion_batch_id`，记录服务、release、错误分类、duration、provider usage 和完成状态。
- 初始 log retention：staging 14 天、production 30 天；根据来源、隐私和排障需求调整，不能无限保留。
- 必须监控 ALB target health / 5xx、ECS task restart / memory、RDS connections / storage、model timeout / rejected requests、预算约束失败与来源过期。
- 安全与 admission metrics 包括 edge blocks、Cognito / Siteverify failures、每类 `429` / `503`、active / reserved slots、token reserved / settled、reconciliation hold age、DB admission latency / lock conflict、跨用户读取拒绝。应用 request ID 与 Cloudflare Ray ID 可关联，但日志不能携带 credentials 或完整聊天。
- 超额或 usage 未知触发告警和 provider circuit breaker；监控用户感知的首个 SSE event latency、完整响应 latency 与拒绝比例，分别记录 cache / mock / real provider，不能将被拒绝请求统计为成功低延迟。
- 启用异步 ingestion 后增加 queue age、retry count、DLQ messages 和审核积压指标。
- 初始告警示例：unhealthy target 持续 5 分钟、连续 3 次 smoke failure、RDS free storage 低于配置阈值；先验证告警能被负责者接收，再根据实际基线调参。
- 公开 health check 不调用付费 model；真实 provider smoke 由受控测试账号低量执行并遵守 live Turnstile 与配额；首轮不安排需要绕过人机 gate 的无人值守收费 health check。

### 8.2 成本控制

首次 `apply` 前记录以下 worksheet；金额必须根据当时 Region、规格与使用量填写，本文不提供未经核算的月费承诺。

| 成本类别 | 必填估算输入 |
| --- | --- |
| Fargate | 三个服务的 vCPU、memory、副本数、运行小时，以及 Service Connect proxy 余量 |
| ALB | 运行小时、LCU、active connections 与 streaming 流量 |
| RDS | engine、instance、Single / Multi-AZ、storage、backup retention |
| 网络 | NAT 数量与小时、NAT data processing、跨 AZ、internet egress、interface endpoints、public IPv4 |
| 存储与分发 | S3 容量 / requests / versions、CloudFront requests / egress、ECR 保留量 |
| 运维与身份 | CloudWatch logs / metrics、Secrets、Cognito、Cognito AWS WAF rules / requests、DNS、Service Connect TLS 的 Private CA / certificates / Secrets / KMS，以及可选 Redis / SQS / Scheduler |
| Cloudflare | 实际 plan、WAF / rate limiting capabilities、Turnstile 配额、日志与附加服务；单独确认计费，不预设全部免费 |
| 外部 API | model tokens / request 次数、Google 或其他数据 API 的实际计费；与 AWS bill 分开记录 |

- 月度 AWS budget 的实际金额由 owner 填写；建议 actual spend 的 50% / 80% / 100% 和 forecast 超额告警。
- AWS Budgets 存在数据更新延迟，告警不是实时 hard cap，也不会自动停止所有资源。[AWS Budgets 文档](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- 应用单独限制 model request rate、concurrency、token limit 和每环境 usage；无法原子预留足够 quota 或达到阈值时拒绝新增付费请求并保留普通目录浏览。AWS cost alert 不承担这项即时控制。
- staging 可按使用时间减少 application task 数；RDS、ALB、NAT 等费用需分别评估，不能把 task 缩为零等同于环境零费用。
- 资源销毁与数据保留有明确 owner；演示结束后的清理必须保留必要 snapshot、raw source evidence 和恢复信息。

### 8.3 恢复要求

- 进入 production 前明确 RPO、RTO、恢复 owner 和允许的数据损失范围；本文不把未测试的数值写成 SLA。
- staging 至少演练一次：恢复 RDS 到新实例 → 更新独立配置 → 验证 schema、已发布菜单、来源引用与一条推荐路径。
- 恢复环境保持隔离，在开放访问前重放恢复点之后的来源撤销 / 删除清单，并重新检查 Source policy、freshness、账号 / session 撤销和未结算 quota reservations；删除清单需独立保留且自身不含已删除内容，不能只依赖同一恢复点内的数据。演练记录实际耗时、可恢复时间点、S3 references 及失败步骤。
- 同时演练误发布菜单的撤回与重新发布；菜单版本修复通常优先通过业务流程完成，不恢复整个数据库。
- production 上线后按变更风险和恢复目标复测；engine upgrade、migration 大改或 storage policy 变化后重新验证。

## 9. 上云前交付与验收

- [ ] 账号、Region、billing owner、月度上限、通知接收者、domain 和 deployment owner 已记录。
- [ ] Terraform plan 展示真实资源及估算；state、OIDC trust、IAM 和 Secrets 已完成最小权限检查。
- [ ] private tasks 能访问所需 AWS services 和一个 model provider；RDS / Redis / AI 无 public access。
- [ ] Cloudflare proxied DNS、Full (strict)、origin IP allowlist、secret header + Host 校验均通过；直连 ALB、伪造 IP / header、DNS-only alias 和 IPv6 旁路均不能访问应用。
- [ ] public `/internal/*`、`/actuator/*` 被阻断；普通用户不能调用 admin 或 ingestion write API。
- [ ] 匿名、未验证邮箱、expired token、disabled account 无法调用任意付费 AI generation；登录正常用户只能读取自己的 conversation / recommendation。
- [ ] 缺少 / 伪造 / expired / replay / wrong-host / wrong-action Turnstile token 均被拒绝，Siteverify outage 不放行；幂等重放不重复消费 token 或 quota。
- [ ] Cognito hosted login 和 User Pool API 的 WAF 独立生效；注册、邮箱确认、登录、password reset 和退出路径均完成 E2E。
- [ ] 隔离 local / CI 的两 Java / 两 AI replicas 下，使用显式 mock auth / challenge 与 barrier mock provider：50 个不同用户并发最多 5 个 active provider attempts；单用户 10 tabs 最多 1 个；20 个相同 idempotency 请求最多 1 次 dispatch。额外请求受控拒绝，quota 不出现负数或超售。
- [ ] Fargate 临时两 Java / 两 AI replicas 的少量 live challenge / provider smoke 验证 routing、shared ledger 和 rolling；公开 staging 始终开启 auth、live Turnstile 与 origin 防护，不能把 local mock load 报为云端完整 load test。
- [ ] mock load 覆盖 cancel、进程 crash、rolling deploy、Redis failure、MySQL admission failure、lease expiry、UTC 换日和 usage 不明；reservation 不重复释放，状态可 reconciliation。完整并发与故障注入在隔离 local / CI 进行；真实第三方仅做少量有额度 smoke，不对第三方做 load test。
- [ ] raw S3 objects 无匿名访问；CloudFront 只分发有权公开的 assets。
- [ ] migration job 成功后才 rollout；release manifest 可定位 image digest、schema 与数据版本。
- [ ] 真实 SSE 分段可见，timeout / cancel / provider failure 有终态，缺少候选时不生成虚构菜品。
- [ ] 推荐预算符合 `MENU_SUBTOTAL`，delivery fee 与 service fee 不计入且分别标注已知 / 未知。
- [ ] Logs 无 credentials，告警可送达；AWS 与 model usage 有分别可见的预算记录。
- [ ] 至少完成一次恢复演练；所有未满足的 production 条件有明确状态，staging 成功不等于 production readiness。

当前待实例化：AWS account、实际 domain、各环境费用上限、engine/runtime 精确版本、task sizing、production 可用性目标和 RPO / RTO。这些事项不阻塞本地 mock provider 与 seed data 的开发。

## 10. 来源核对说明

通用 AWS、GitHub 与 Terraform 资料于 2026-10-28 核对；Service Connect TLS、ALB health check 和 proxy timeout 于 2026-10-29 再次核对。Cloudflare origin protection、header rules、Full (strict)、Turnstile validation / testing、rate limiting 与 Cognito AWS WAF 于 2026-11-05 核对。具体 topology、timeout、retention、quota、secret header 与部署阶段属于本项目设计选择；首次部署仍需按实际 plan、runtime 和配置验证。本文的验收项仍待执行，文档更新不代表已创建云资源或证明防护生效。
