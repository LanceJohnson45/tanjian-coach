# Agent Skill 授权码（Agent Keys）API 合约

日期：2026-11-02
状态：**已定稿，后端按此实现**
前置阅读：`docs/api/2026-10-11-auth-email-login.md`（以下简称「M5 合约」）；`docs/api/2026-09-22-flutter-app-api.md`（以下简称「主合约」）。

## 0. 目标与范围

- 用户可在 App 账户页生成**个人授权码（agent key）**，配置到 AI agent（ZCode / Claude Code 等 skill 体系，交付物 `skills/tanjian-coach/`）后，agent 以 `Authorization: Bearer <key>` 调用本服务：
  - **读训练数据**：`/api/workout-plans*`、`/api/workout-sessions*`、`/api/workout-sets*`（原本就要求 Bearer 的全部私有端点）；
  - **读公开库**：exercises / knowledge-items / articles / workout-templates 等（匿名端点，无需授权码即可用）；
  - **写训练计划**：`POST/PUT/DELETE /api/workout-plans`、`POST /api/workout-templates/{id}/apply`。
- **一码全权**（用户已确认）：agent key 等价于其持有者的完整数据读写能力，无 scope 区分；用户随时可吊销。
- key 为不透明随机串：`tj_` + `secrets.token_urlsafe(32)`（共 46 字符）。明文**只在创建响应中出现一次**；库里只存 `sha256(key)`。
- `tj_` 前缀仅是 UX 标识（用户/日志可识别码类型）；**鉴权分发不依赖前缀**（见 §2，先查 `auth_tokens` 再查 `agent_keys`，杜绝前缀碰撞）。
- 零新增 Python 依赖。

## 1. 端点（管理端点，全部要求 **session token**，agent key 不可调用，见 §2）

### 1.1 POST /api/agent-keys

请求体：

```json
{ "name": "我的 ZCode" }
```

- `name` 必填，1–40 字符（去首尾空白后），用于用户辨识（如「我的 ZCode」「家里电脑」）。
- 每用户 active key 数量上限 `AGENT_KEY_MAX_PER_USER`（缺省 20），超限 → 409 `conflict`。
- 生成 `key = "tj_" + secrets.token_urlsafe(32)`；插入 `agent_keys`（`key_hash = sha256(key)`）。

响应 201（`key` 明文唯一一次出现）：

```json
{
  "id": "uuid",
  "name": "我的 ZCode",
  "key": "tj_<43-urlsafe-chars>",
  "created_at": "2026-11-02T12:00:00Z"
}
```

### 1.2 GET /api/agent-keys

响应 200（本人 keys，`created_at desc`；**永不返回 key/hash**）：

```json
{
  "items": [
    { "id": "uuid", "name": "我的 ZCode", "created_at": "…", "last_used_at": "…|null" }
  ]
}
```

- 条目 ≤ `AGENT_KEY_MAX_PER_USER`，**不分页**（非 `Page` 信封，同 `workout-templates` 列表的 `{items}` 先例）。
- `last_used_at` 为该 key 最近一次被用于鉴权的时间（节流更新，见 §2），null = 从未使用。

### 1.3 DELETE /api/agent-keys/{id}

- 物理 `delete from agent_keys where id = %s and user_id = %s`，删除行数 0（不存在**或非本人**）→ 404 `not_found`；成功 → 204 空体。
- 不幂等：删除后重复 DELETE → 404（非本人资源不泄露存在性，与主合约 404 语义一致）。
- 吊销即刻生效：被删 key 的下一次请求 → 401。

错误码汇总（本节即权威，补进主合约 §3.3 实现层）：

| code | HTTP | 触发 |
|------|------|------|
| `unauthorized` | 401 | Bearer 缺失/无效/过期；**agent key 调本节端点** |
| `validation_error` | 422 | name 缺失/空/超 40 字符（主合约信封） |
| `conflict` | 409 | active key 数量达上限（`details.limit` 给上限值） |
| `not_found` | 404 | DELETE 目标不存在或非本人 |

## 2. 鉴权解析（改造 `app/deps.py`）

### 2.1 `get_user_id`（签名与返回值不变，现有路由零改动）

1. 解析 `Authorization: Bearer <token>`（同 M5 合约 §2）。
2. `sha256(token)` 查 `auth_tokens`（带 `expires_at > now()`）→ 命中：按现有节流逻辑 touch `auth_tokens.last_used_at`，返回其 `user_id`。
3. **未命中再查 `agent_keys`**（`where key_hash = %s`，无过期条件）→ 命中：同款 1h 节流 touch `agent_keys.last_used_at`，返回其 `user_id`。
4. 两表都未命中 → 401 `unauthorized("Invalid or expired session token")`（文案不改，不向探测方泄露命中的是哪张表）。

顺序查找（而非按前缀分发）是刻意的：`token_urlsafe` 字符集理论上可产生 `tj_` 开头的串，顺序查找让任何 session token 永远优先命中自己的表，零碰撞风险。

### 2.2 新增 `get_session_user_id`（仅 §1 管理端点使用）

- 只执行上述第 1、2 步（仅 `auth_tokens`）；agent key 或无效 token → 401。
- 语义：**agent key 不能创建/吊销 agent key**（防一把泄漏的 key 自我繁殖），也不能作为 `/api/auth/email-change*` 的凭证（M5 合约端点改挂此依赖）。
- `/api/auth/me` 保持 `get_user_id`（agent key 可用——skill 首步自检需要）；`/api/auth/session` DELETE 维持现状不动（agent key 调它按 hash 查 `auth_tokens` 删不到行，返回 204，无害；不额外特判）。

### 2.3 agent key 能力边界（权威表述）

| 端点 | session token | agent key |
|---|---|---|
| workout-plans / sessions / sets 全部读写 | ✅ | ✅ |
| workout-templates apply | ✅ | ✅ |
| GET /api/auth/me | ✅ | ✅（skill 自检） |
| DELETE /api/auth/session | ✅ | 204 无效果（删不到 session 行） |
| §1 管理端点 / email-change* | ✅ | ❌ 401 |

## 3. 数据库（`backend/migrations/005_agent_keys.sql`，幂等，风格同 003）

```sql
create table if not exists agent_keys (
  id uuid primary key default uuid_generate_v4(),
  key_hash text not null unique,
  user_id uuid not null references users(id) on delete cascade,
  name text not null,
  created_at timestamptz not null default now(),
  last_used_at timestamptz
);
create index if not exists idx_agent_keys_user on agent_keys(user_id);
```

- 明文 key 永不落库（同 `auth_tokens` 口径）；`sha256` 仅防库泄露后的离线比对。
- 行随 user 级联删除。迁移挂进既有三通路：`tests/conftest.py MIGRATIONS`、`scripts/dev_setup.sh`、`deploy/setup-host-pg.sh`（Dockerfile COPY 整目录无需改）。

## 4. 环境变量（沿用 db.py 的 os.environ 直读风格）

| 变量 | 缺省 | 说明 |
|------|------|------|
| `AGENT_KEY_MAX_PER_USER` | `20` | 每用户 active key 上限 |

## 5. 测试要求（pytest，新文件 `backend/tests/test_agent_keys.py`）

1. 管理：创建 201（key 以 `tj_` 开头、46 字符、响应结构）、列表 200（不含 key/hash、created_at desc）、DELETE 204 后列表消失、DELETE 不存在的 id 404、DELETE 他人 key 404（两用户隔离）。
2. 上限：造满 `AGENT_KEY_MAX_PER_USER` 后再创建 → 409（测试里 `monkeypatch.setenv` 调小上限）。
3. **agent key 走通业务面**：用创建返回的 key 作为 Bearer —— `GET /api/workout-plans` 200、`POST /api/workout-plans` 201 且归属该用户、`GET /api/auth/me` 200 返回本人 email。
4. 边界：agent key 调 `POST /api/agent-keys` → 401；agent key 调 `DELETE /api/auth/session` → 204 且原 session token 仍可用；吊销后的 key 再调任何端点 → 401；anon 调管理端点 → 401；name 空/41 字符 → 422。
5. `last_used_at`：创建后列表为 null；用 key 打一次 `me` 再列表非空。
6. 全量回归绿 + 主控复跑通过才算完成。

## 6. Flutter 对接要点（前端实现依据，后端不实现本节）

- 账户页 `_actionsCard` 新增入口「Agent 授权码」（仅登录态渲染，天然守卫），push 新页。
- 新页三态：mock 提示卡 / 游客登录引导 / 已登录列表 + 生成。
- 生成两步弹层：输入备注名 → 成功后**一次性明文弹层**（`SelectableText` + 复制按钮 + 「关闭后无法再次查看」警示）；明文不落盘、不打印日志。
- 吊销二次确认（AlertDialog）；失败态 ErrorBox 重试；错误文案走 `friendlyApiError`。

## 7. Skill 接入规范（`skills/tanjian-coach/` 的权威依据）

- **配置**：`TANJIAN_API_BASE`（如 `https://gym.4bug.eu.org`，本地联调 `http://127.0.0.1:8000`）+ `TANJIAN_AGENT_KEY` 两个环境变量；首步 `GET /api/auth/me` 自检配置。
- **工作流 A（分析训练）**：me → `GET /workout-plans`（现状基线）→ `GET /workout-sessions?from=&to=&page_size=100` 翻页（信封 `total` 判断是否继续）→ 按需 `GET /workout-sessions/{id}` 取 sets → 结合 `GET /exercises`、`GET /knowledge-items` 检索动作要领与知识依据。
- **工作流 B（创建计划）**：红线——`exercise_id` **必须来自 `GET /api/exercises` 的真实检索结果，禁止编造 UUID**（服务端会 422 `invalid_exercise_ids`）；再 `POST /api/workout-plans`（schema 见 `app/schemas/requests.py:WorkoutPlanCreate`，phase ∈ warmup/main/cooldown）；或从 `GET /api/workout-templates` 选模板 apply。
- **纪律**：key 不写入任何产出物/日志/commit。

## 8. 变更流程

本合约不新增领域模型（`docs/01` 不动）；实现时回写 `docs/04-database.md`（agent_keys 表）与 `docs/05-api.md`（端点清单）。
