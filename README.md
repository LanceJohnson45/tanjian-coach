# tanjian-coach skill

让 AI 助手（ZCode / Claude Code 等支持 skill 的 agent）接入你在「谭健」App 里的训练数据：读训练记录与计划、查动作库与知识库、分析训练表现、直接创建训练计划写回你的账号。

API 合约权威文档：`docs/api/2026-11-02-agent-skill-access.md`（本仓库内）。

## 安装（一次性）

把本目录复制到你的 agent 的 skills 目录：

```bash
# ZCode（全局）
git clone https://github.com/LanceJohnson45/tanjian-coach.git cp -r skills/tanjian-coach ~/.zcode/skills/cp -r skills/tanjian-coach ~/.zcode/skills/ cp -r tanjian-coach ~/.zcode/skills/

# 或 Claude Code 类 agent 的对应 skills 目录
```

## 获取授权码（App 内）

1. 打开「谭健」App → 首页右上角**头像**
2. 账户页 → **Agent 授权码**
3. 点「生成第一个授权码」→ 起个备注名（如「我的 ZCode」）
4. **立即复制**——明文只显示这一次，关闭弹层后无法再查看
5. 不再使用时回到同一页面**吊销**，即刻生效

## 配置环境变量

```bash
export TANJIAN_API_BASE="https://gym.4bug.eu.org"   # 你部署的服务地址
export TANJIAN_AGENT_KEY="tj_xxxxxxxxxxxxxxxxxxxxx"  # 上一步复制的授权码
```

建议写进 shell profile 或 agent 的密钥管理方式里（key 等同你账号的完整读写权限，不要提交进仓库/截图外发）。

## 验证

```bash
curl -s -H "Authorization: Bearer $TANJIAN_AGENT_KEY" "$TANJIAN_API_BASE/api/auth/me"
```

返回你的邮箱即配置成功。之后直接对 AI 助手说：

- 「分析一下我最近的训练」
- 「帮我排一个下周的下肢计划，根据我的历史重量」

## 安全说明

- 授权码（`tj_` 开头）在服务端只存 sha256 哈希，吊销即刻生效
- 一码全权（读训练数据 + 读写计划），每用户上限 20 个，随时在 App 内吊销
- 授权码不能管理授权码（生成/吊销仅限 App 内登录会话），不能改账户邮箱
