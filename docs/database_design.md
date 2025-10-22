# 数据库设计说明

## 总体概览
New API 采用 GORM 作为 ORM，支持 MySQL、PostgreSQL 以及 SQLite 三种存储引擎。`model.InitDB` 会根据 `SQL_DSN` 和 `LOG_SQL_DSN` 初始化业务库 (`DB`) 与日志库 (`LOG_DB`)，默认使用同一库。当启用独立日志库时，可将高并发的消费日志写入独立实例，降低业务库压力。所有结构体通过 `AutoMigrate` 自动建表，因此新增字段需保持向后兼容并关注默认值。时间字段统一使用 `int64` 秒级时间戳以兼容多数据库驱动。

## 核心业务表

### 用户与鉴权相关

#### `users`
| 字段 | 数据类型 | 说明与业务用途 | 存储理由 / 可能改进 |
| --- | --- | --- | --- |
| `id` | `int` | 主键 | 传统自增整型，满足当前规模 |
| `username` | `varchar(20)` 唯一索引 | 登录名、也是日志归属依据 | 长度限制可避免异常输入；若需多语言用户名可放宽限制 |
| `password` | `varchar` | Bcrypt 哈希 | 字符串存储哈希，避免二进制兼容问题 |
| `role`/`status` | `int` | 权限与账号状态 | 小范围枚举，使用整型提升比较效率 |
| `quota`/`used_quota` | `int` | 累计可用额度/已用额度 | 与消费逻辑一致；若需要更大阈值可升级为 `bigint` |
| `access_token` | `char(32)` | 管理端访问令牌 | 固定长度便于索引；考虑将字符串改为随机字节并 Base64 增强熵 |
| `setting` | `text` (JSON 字符串) | 个性化开关、侧边栏配置 | 文本 JSON 兼容 SQLite；如仅支持 MySQL/PG，可改 JSON 原生列并增加虚拟列索引 |

`service/user` 模块在登录、配额扣减时频繁读取 `quota` 与 `status`，使用整型可直接进行原子加减。缓存层 (`model/user_cache.go`) 将常用字段写入 Redis Hash，保持热路径高效。

#### `tokens`
| 字段 | 数据类型 | 说明 | 存储理由 / 改进 | 
| --- | --- | --- | --- |
| `key` | `char(48)` 唯一索引 | API Key（`sk-` 前缀） | 固定长度便于脱敏，Redis 中以 HMAC 形式缓存 |
| `remain_quota`/`used_quota` | `int` | 剩余额度/已用额度 | 与用户额度保持相同单位；若渠道计费可能超出 2^31 建议升级 `bigint` |
| `allow_ips` | `text` | 允许访问的 IP 列表，换行分隔 | 字符串解析降低建表成本；如需要更灵活控制，可拆分独立 IP 绑定表 |
| `model_limits` | `varchar(1024)` | 受限的模型列表 | 通过逗号存储，便于与前端表单映射；若后续需要复杂限制可改 JSON |

业务逻辑在 `service/quota.go` 里将 Token 与 User 的额度联合判断，字段采用整型方便直接做数学运算。

#### Passkey & 2FA
- `passkey_credentials`：主要字段如 `credential_id`、`public_key` 都使用 Base64 字符串保留原始字节，可跨数据库且不丢精度；布尔位标记 WebAuthn Flags。可考虑在 MySQL/PG 上使用 `VARBINARY` 以减少空间占用。
- `two_fa`、`two_fa_backup_codes`：`secret` 与备份码哈希存储为 `varchar(255)`，允许使用任意哈希算法输出。`locked_until`、`used_at` 使用 `timestamp` 以配合锁定策略。

### 渠道、模型与路由

#### `channels`
| 字段 | 数据类型 | 说明 | 存储理由 / 改进 |
| --- | --- | --- | --- |
| `type`/`status` | `int` | 渠道类型与启用状态 | 枚举映射常量，整型便于比较 |
| `key` | `text` | 上游 Access Key，可存储多行 | 字符串允许多 Key；多 Key 场景由 `ChannelInfo` 管理 |
| `models` | `text` | 渠道支持模型逗号串 | 简单高效；若模型数量增长可建关联表 |
| `channel_info` | `json` | 多 Key 配置、轮询索引 | JSON 存储动态结构，配合 `ChannelInfo` 自定义 Scanner；跨引擎兼容 |
| `model_mapping`、`param_override`、`header_override` | `text` | 按模型覆盖配置 | 使用 JSON 字符串封装，便于前端一次性提交；若迁往 PostgreSQL，可转 JSONB 便于查询 |
| `balance` | `float64` | 渠道余额美元 | 浮点便于快速更新；如需精确计费建议改用 `decimal` |

`service/channel.go` 会根据 `weight`、`priority` 与 `channel_info` 做二次调度。`channel_info.MultiKeyPollingIndex` 通过 JSON 存储轮询状态并在内存缓存中更新，避免额外表结构。

#### `abilities`
三字段复合主键 (`group`,`model`,`channel_id`) 标记渠道能力。`weight` 与 `priority` 决定路由顺序，整型在算法中开销最低。若后续需要记录阈值或失败率，可扩展字段。

#### `models` / `vendors`
- `models`：`model_name` 唯一索引，同时维护 `description`、`tags`、`endpoints` 等信息。`name_rule` 保存匹配方式，可在 `pricing` 逻辑中实现前缀/后缀匹配。`endpoints` 作为文本 JSON 储存，方便扩展。
- `vendors`：维护供应商元数据，`icon` 使用 `varchar(128)` 可直接引用前端 icon 名称。配合 `pricing` 动态构建模型展示。

#### `prefill_groups`
`items` 使用自定义 `JSONValue` (兼容 `[]byte`/`string`) 以 JSON 数组存储常用组合（如模型组、端点组）。`type` 字段使其可复用。该设计平衡了扩展性与简单性；在 PostgreSQL 上可改 `jsonb` 并增加 GIN 索引支持包含查询。

### 额度、计费与交易

#### `top_ups`
| 字段 | 类型 | 说明 | 备注 |
| --- | --- | --- | --- |
| `amount` | `int64` | 实际支付金额（分） | 使用整型避免浮点误差 |
| `money` | `float64` | 折算成人民币/美元金额 | 目前仅用于展示，可考虑改 `decimal(10,2)` |
| `status` | `varchar` | `pending/success` 等 | 与第三方回调状态保持一致 |

事务 (`Recharge`) 中以 `FOR UPDATE` 锁定订单，并将充值额度折算为 `quota`。订单号 `trade_no` 采用唯一索引确保幂等。

#### `redemptions`
兑换码信息。`key` 使用 `char(32)` 保证对齐生成器输出，`quota` 为 `int` 与消费逻辑一致。`expired_time` 采用时间戳，可对过期码批量软删除。若需要多次使用的卡券，可新增总次数与剩余次数字段。

#### `quota_data`
小时粒度的使用统计。`token_used`、`quota` 均为 `int`，满足当前按小时聚合的范围。`username` 与 `model_name` 均建联合索引，配合后台统计图表。由于是长期累积表，建议按 `created_at` 分区或定期归档。

### 日志与异步任务

#### `logs`
`created_at` 以 `int64` 时间戳保存，便于跨数据库排序。内容字段 `other` 存储 JSON 字符串（例如错误上下文、管理员信息），在返回前会剔除敏感项。系统支持将日志写入独立数据库以解耦业务压力。若使用 PostgreSQL，可把 `other` 改为 `jsonb` 并增加键值索引提升查询能力。

#### `tasks`
| 字段 | 数据类型 | 说明 | 理由 |
| --- | --- | --- | --- |
| `task_id` | `varchar(191)` | 第三方任务编号 | 兼容 Suno/Midjourney 等不同 ID 长度 |
| `platform` | `varchar(30)` | 任务平台枚举 | 字符串直接对应常量，便于前端显示 |
| `status` | `varchar(20)` | `QUEUED/SUCCESS` 等 | 直接与 webhook 状态匹配 |
| `properties` | `json` | 任务扩展字段（如输入内容） | 使用 `Properties` 自定义 Scanner/Valuer，保持数据结构灵活 |
| `data` | `json` | 第三方返回原始数据 | Raw JSON 保留细节，可支持多种接口 |

`service/task.go` 会根据 `status` 与 `progress` 轮询更新，`json.RawMessage` 便于存储异构响应。若需要对 `data` 做检索，可考虑拆分核心字段到显式列。

#### `midjourney`
专门记录 MJ 绘图任务，字段与任务表相似，但 `buttons`、`properties` 使用字符串保留按钮状态。由于多语言提示较长，`prompt` 字段使用 `text`。若未来需要对 `prompt` 做全文检索，可在 MySQL 上启用 FULLTEXT。

### 系统配置与缓存说明

#### `options`
键值对配置表，`key` 为主键，`value` 使用字符串统一存储。业务启动后先写入内存映射，再通过数据库覆盖。该设计保证兼容多种类型但缺乏类型约束，建议对关键配置（如倍率、布尔开关）增加类型校验或拆分表结构。

#### 缓存回写策略
`user_cache`、`token_cache` 均未建表，而是通过 Redis Hash 缓存热数据。数据库字段 `Setting`、`ModelLimits` 等因仍需持久化，采用 TEXT/JSON。消费链路在 Redis 未命中时回源数据库并回写缓存，保证与业务库的一致性。

## 数据类型选择分析
1. **整型额度与计时**：配额、令牌剩余额度广泛使用 `int`。与 `service/quota.go` 的数学运算匹配，且与 Redis `HIncrBy` 兼容。若接入高额度渠道（>20 亿），需将相关字段统一升级为 `bigint` 并同步 Redis 脚本。
2. **浮点金额**：`channels.balance`、`top_ups.money` 当前使用 `float64`，方便快速写入但存在舍入风险。若结算精度要求提高，建议改为 `decimal(18,4)` 或拆分 `amount_cent` 整型。
3. **JSON/TEXT 混用**：为兼容 SQLite，部分 JSON 字段使用 `text`（如 `users.setting`、`channels.model_mapping`）。在 MySQL/PG 环境中可以改用原生 JSON/JSONB 并新增虚拟列或 GIN 索引以提升查询性能。
4. **时间戳统一**：所有时间字段采用秒级 `int64`，跨数据库计算简单，但在报表层需转成人类可读格式。若迁移至 PostgreSQL，可考虑改 `timestamptz` 并由 GORM 配置自动转换。

## 改进建议与替代方案
1. **额度字段统一提升**：`users.quota`、`tokens.remain_quota` 与 `quota_data.quota` 可统一改为 `bigint`，避免未来多币种或高倍率模型导致上限不足。
2. **Key/IP 的结构化存储**：`channels.key`、`tokens.allow_ips` 当前以换行字符串存储，解析成本随数据量上升。可新增 `channel_keys`、`token_allowed_ips` 从表，便于按 Key 状态建索引，并支持按 IP 快速搜索。
3. **JSON 字段索引化**：在 MySQL 8 / PostgreSQL 中，可将 `channel_info`、`param_override` 等迁移至 JSON 原生类型，并通过虚拟列创建索引（例如 `channel_info->'$.multi_key_mode'`）。
4. **日志归档策略**：`logs` 表增长快速，建议配合独立库使用分区或冷热分离（定期将旧日志导出至对象存储）。
5. **金额精度**：与支付相关的 `top_ups.money`、`channels.balance` 建议改为 `DECIMAL` 或拆成整数货币单位，避免浮点误差。
6. **配置类型约束**：`options` 表可增加 `type`/`json_schema` 字段，或拆分出倍率、布尔、文本等不同配置表，降低错误配置风险。
7. **任务数据规范化**：对常用查询字段（如 `Task.Data` 中的音视频 URL）抽离到独立列，避免每次解析 JSON。

通过以上分析，现有设计在多数据库兼容、快速迭代方面具备弹性。后续可根据业务规模重点优化额度字段与 JSON 存储策略，以获得更好的查询性能与数据一致性。
