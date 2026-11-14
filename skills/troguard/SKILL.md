---
name: troguard
description: Check print-on-demand, apparel, and cross-border ecommerce images for copyright and TRO infringement risk with TroGuard. Use when the user mentions TRO, TroGuard, 侵权检测, 版权检测, 印花检测, 图案风险, TRO 案件库, batch image checks, or TROGUARD_API_KEY, or asks whether artwork is safe to list.
metadata:
  author: TroGuard
  version: 1.0.0
---

# TroGuard 图片侵权检测

用 TroGuard 的公开检测接口，对印花、图案、主图、详情页和包装素材做版权与 TRO 风险筛查。输出风险等级、证据来源和案件线索；结果是筛查线索，不是法律意见。

## 何时使用

- 用户要检查一张或多张图片是否有 TRO / 版权风险
- 用户提供印花、插画、角色或品牌元素，询问能不能上架
- 用户要做批量自查、汇总高风险图片
- 用户提到 API Key、盾石、TRO 案件库

## 前置条件

1. API Key：在 <https://app.troguard.com/team/api-keys> 创建；完整 Key 只显示一次。
2. 团队盾石：1 盾石/张，失败任务退还；余额不足返回 402 `insufficient_credits`。
3. 环境变量：`TROGUARD_API_KEY` 必填；`TROGUARD_API_BASE` 可选，默认 `https://app.troguard.com`。

不要在聊天里索要或回显完整 Key。让用户自己在终端或 CI secret 里设置。

## 安装

```bash
npx skills add affine-tech/troguard-skills@troguard
```

支持 Claude Code、Codex / OpenCode、Cursor 等兼容 Agent Skills 标准的工具。

## 首选：运行脚本

```bash
export TROGUARD_API_KEY='trk_...'
python3 scripts/tro-check.py ./pattern-a.png ./pattern-b.jpg --name "上新自查"
```

脚本路径相对于本 skill 目录。它会：

1. 读取 1–200 张图片，批量提交到 `POST /api/public/tro-check/batch`
2. 轮询 `GET /api/public/tro-check/{batch_id}` 直到所有任务结束
3. 输出批次摘要、风险计数和高风险图片

常用参数：`--json` 输出完整批次、`--env-file ~/.config/troguard.env` 读取 Key、`--timeout 900` 调整等待时间、`--no-proxy` 绕过代理。

## 手动调用

```http
POST https://app.troguard.com/api/public/tro-check/batch
X-API-Key: trk_...
Content-Type: application/json

{"name":"agent-check","images":[{"filename":"print.png","image_base64":"..."}]}
```

响应里的 `data.id` 是 `batch_id`；带同一个 `X-API-Key` 轮询：

```http
GET https://app.troguard.com/api/public/tro-check/{batch_id}
X-API-Key: trk_...
```

单批 1–200 张，单张解码后 ≤10MB。可选 `Idempotency-Key: <8-64 位字母、数字或 -_.:>`，重复提交返回同一批次，不重复扣盾石。

## 结果怎么读

- `risk_level`：`critical` / `high` / `medium` / `low` / `unknown`
- 同图命中 TRO 案件库会列为 `critical`；优先展示权利人、案号和案件图
- 每条结果包含 `summary`、`evidence[]`、`confidence` 和 `advice`
- 汇总时先列 `critical` 和 `high`，再列其余；提醒用户人工核对来源和授权

## 错误处理

| 状态 / code | 含义 | 处理 |
|---|---|---|
| 401 `unauthorized` | Key 无效、已吊销、用户或团队不可用 | 到 Agent Skill 页面重新创建 |
| 402 `insufficient_credits` | 盾石不足 | 充值后重试 |
| 410 `aipay_deprecated` | 没有 API Key，402 按次支付已停用 | 改用 API Key |
| 400 `invalid_input` | 图片格式/大小不对，或参数非法 | 修正请求 |
| 429 `too_many_requests` | 触发限流 | 降低并发或稍后重试 |
| TLS reset / `curl: (35)` | 到 `app.troguard.com:443` 的握手被重置 | 用脚本（Python/OpenSSL）或 TLS 1.3 客户端；换网络 |

完整字段、限流和接口说明见 `references/api.md`。

## 安全

- Key 只在创建时显示一次，数据库只存哈希。
- 不要把 Key 写入仓库、日志、截图或聊天；泄露后立即在门户吊销并重建。
- 只上传有授权的图片；检测记录按 TroGuard 隐私政策保留。

## 边界

- 只做图片侧风险筛查，不提供法律意见、诉讼代理或 TRO 应对方案。
- 不承诺覆盖所有权利来源；未命中不代表没有风险。
- 商标注册、专利、非图片类知识产权不在能力范围内。
