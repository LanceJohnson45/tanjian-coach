---
name: tanjian-coach
description: 谭健（TANJIAN）私人健身教练数据接入。当用户想「分析我的训练数据 / 看看我最近的训练情况 / 帮我排一个训练计划 / 我这个动作练得怎么样 / 根据我的历史安排下周训练」等涉及健身训练数据读写与计划编排的任务时使用本 skill。数据来源是用户自部署的谭健健身服务（训练记录、动作库、知识库、计划、模板），凭用户个人授权码访问。
---

# 谭健教练（tanjian-coach）

接入「谭健」健身服务的用户数据：读训练记录与计划、查动作库与知识库、分析训练表现、创建训练计划写回用户账号。

## 0. 配置（先做这一步）

需要两个环境变量：

```bash
export TANJIAN_API_BASE="https://gym.4bug.eu.org"   # 用户自部署的服务地址（本地联调 http://127.0.0.1:8000）
export TANJIAN_AGENT_KEY="tj_..."                    # 用户的个人授权码
```

**没有 key 时**：引导用户打开手机 App「谭健」→ 首页右上角头像 → 账户 → **Agent 授权码** → 生成并复制。不要继续，等用户提供。

配置自检（每次会话先跑一次）：

```bash
curl -s -H "Authorization: Bearer $TANJIAN_AGENT_KEY" "$TANJIAN_API_BASE/api/auth/me"
```

返回 `{"id": "...", "email": "..."}` 即配置正确。401 → key 错或已被吊销，让用户重新生成。网络不通 → 确认 TANJIAN_API_BASE。

所有请求统一错误信封 `{"error": {"code", "message", "details"}}`。

## 1. 端点速查

| 用途 | 方法+路径 | 认证 |
|---|---|---|
| 我是谁 / 自检 | GET `/api/auth/me` | Bearer |
| 我的计划列表（含动作项） | GET `/api/workout-plans` | Bearer |
| 计划详情 / 创建 / 全量替换 / 删除 | GET / POST / PUT / DELETE `/api/workout-plans[/{id}]` | Bearer |
| 训练记录列表（`from`/`to`/`exercise_id` + 分页） | GET `/api/workout-sessions` | Bearer |
| 单次训练详情（含全部组数记录） | GET `/api/workout-sessions/{id}` | Bearer |
| 从预制模板生成计划（可同时开练） | POST `/api/workout-templates/{id}/apply` | Bearer |
| 动作检索（`q`/`muscle_id`/`equipment_id`/`difficulty`/`movement_pattern`/`series`） | GET `/api/exercises` | 匿名 |
| 动作详情（肌群/器械/媒体/知识点） | GET `/api/exercises/{id}` | 匿名 |
| 知识点检索（`q`/`item_type`/`exercise_id`/`muscle_id`/`tag`） | GET `/api/knowledge-items` | 匿名 |
| 文章（`q`/`tag`/`exercise_id`） | GET `/api/articles` | 匿名 |
| 预制训练模板目录 | GET `/api/workout-templates` | 匿名 |

分页约定：`page`（≥1）+ `page_size`（1–100），信封 `{"items", "page", "page_size", "total"}`——**翻页看 `total`**。时间一律 ISO8601 UTC（`2026-11-02T00:00:00Z`）。

## 2. 工作流 A：分析训练

```bash
# ① 身份与邮箱（确认在分析谁）
curl -s -H "Authorization: Bearer $TANJIAN_AGENT_KEY" "$TANJIAN_API_BASE/api/auth/me"

# ② 现有计划（用户当前的训练安排）
curl -s -H "Authorization: Bearer $TANJIAN_AGENT_KEY" "$TANJIAN_API_BASE/api/workout-plans"

# ③ 训练记录：先拉近一段时间的列表（例：近 8 周）
curl -s -H "Authorization: Bearer $TANJIAN_AGENT_KEY" \
  "$TANJIAN_API_BASE/api/workout-sessions?from=2026-09-07T00:00:00Z&page_size=100"

# ④ total > 已取条数则继续翻 page=2,3...
# ⑤ 需要组数细节（重量/次数/RPE/时长）时逐条拉详情
curl -s -H "Authorization: Bearer $TANJIAN_AGENT_KEY" \
  "$TANJIAN_API_BASE/api/workout-sessions/{session_id}"
```

会话状态：`in_progress` 进行中 / `completed` 已完成 / `abandoned` 已放弃。组记录里计时动作用 `actual_duration_sec`，次数动作用 `actual_reps` + `weight_kg`。

分析时结合知识库增强依据（匿名即可访问）：

```bash
# 某动作的要领/常见错误/教练提示
curl -s "$TANJIAN_API_BASE/api/knowledge-items?exercise_id={exercise_id}"
curl -s "$TANJIAN_API_BASE/api/exercises/{exercise_id}"
```

给出分析结论时要引用真实数据（日期、容量、次数趋势），不要泛泛而谈。

## 3. 工作流 B：创建训练计划

**红线：`exercise_id` 必须来自 `GET /api/exercises` 的真实检索结果，禁止编造 UUID**（服务端会以 422 `invalid_exercise_ids` 拒绝，且编造的 id 即使偶然撞上也挂错动作）。

```bash
# ① 按目标检索候选动作（支持中文/拼音名、肌群、模式）
curl -s "$TANJIAN_API_BASE/api/exercises?q=深蹲&page_size=50"

# ② 组装计划并创建（写回用户账号，App 内立即可见）
curl -s -X POST "$TANJIAN_API_BASE/api/workout-plans" \
  -H "Authorization: Bearer $TANJIAN_AGENT_KEY" -H "Content-Type: application/json" -d @- <<'JSON'
{
  "name": "下肢力量周计划",
  "goal": "strength",
  "difficulty": "intermediate",
  "estimated_duration_min": 60,
  "exercise_items": [
    {
      "exercise_id": "<从检索结果拿的真实 UUID>",
      "phase": "warmup",
      "sort_order": 0,
      "target_sets": 2, "target_reps_min": 10, "target_reps_max": 12, "rest_sec": 60
    },
    {
      "exercise_id": "<...>",
      "phase": "main",
      "sort_order": 1,
      "target_sets": 4, "target_reps_min": 6, "target_reps_max": 8,
      "target_weight_kg": 80, "rest_sec": 180, "tempo": "3-1-1"
    }
  ]
}
JSON
```

字段规范：

- `goal`：`strength` / `hypertrophy` / `endurance` / `mobility` / `fat_loss` / `general_fitness`（可省）
- `difficulty`：`beginner` / `intermediate` / `advanced`（可省）
- `phase`：`warmup` / `main`（缺省）/ `cooldown`；`sort_order` 从 0 起按执行顺序
- 次数动作用 `target_reps_min`/`target_reps_max`；计时动作（平板支撑等）用 `target_duration_sec`/`target_duration_max_sec`，二选一
- 修改既有计划 = GET 该计划 → 改后整包 PUT（无 PATCH）

替代路径——基于服务端预制模板（体系化课程，含知识来源）：

```bash
curl -s "$TANJIAN_API_BASE/api/workout-templates"          # 挑模板
curl -s -X POST "$TANJIAN_API_BASE/api/workout-templates/{template_id}/apply" \
  -H "Authorization: Bearer $TANJIAN_AGENT_KEY" -H "Content-Type: application/json" \
  -d '{"start_session": false}'
```

创建成功响应 201 含完整计划（`id`、全部 `exercise_items`）。把计划要点（动作、组次、负荷）讲给用户听，并提醒可在 App「计划」页查看。

## 4. 纪律

- **key 不落任何产出物**：不写进报告/文件/commit/日志；curl 示例一律用 `$TANJIAN_AGENT_KEY` 引用。
- 授权码等同用户完整读写权限，只执行用户明确要求的操作；创建计划前把方案先讲给用户确认。
- 用户数据仅用于当次对话的分析任务，不做跨用户推断。
- `tj_` 前缀即授权码标识；它**不能**管理授权码本身（生成/吊销只能在 App 内做）。
