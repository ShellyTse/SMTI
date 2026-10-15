# SMTI Growth Lab — Tracking V1 (待验收)

## 0. 环境隔离与验收（2026-10-15 更新）

代码使用明确主机名白名单，不再把一切非生产域名默认为预览环境：

- `www.smtitest.top`、`smtitest.top`、`smti-production.up.railway.app` → 生产 Property，Measurement ID `G-GRM45XJCSH`。
- `smti-smti-pr-1.up.railway.app` → 独立 Staging Property，Measurement ID `G-Q5WKDCB2YN`，开启 debug_mode。
- 其他未知主机、`localhost` → 不加载 GA4，不发送事件。以后需要新的预览地址时，应显式加入测试白名单；切勿加入正式域名。

**验收结果：**
- JavaScript 语法解析与7种域名分发模拟：PASS，未知域名 fail closed。
- Railway PR Environment `SMTI-pr-1` 实际已部署并在线；浏览器控制台显示 `environment: preview` 和正确测试 Measurement ID。
- GA4 Staging DebugView 已实际收到 `test_start`、`answer_progress`、`test_complete`、`result_view`、`share_click`、`share_copy_success`、`retest_click`、`share_landing`。
- `answered_count` 观测到 1/4/7/11/14；`share_landing` 的 `share_method` 为 `direct_link`。
- 已在 GA4 Staging 创建事件范围自定义维度 `answered_count`、`share_method`、`test_version`。
- 浏览器 Network 中正式与测试 GA4 endpoint 的请求对照尚未独立核验；生产发布后应检查新代码已加载、页面功能和正式 Property 数据。

**发布边界：** 只合并此 PR 的埋点代码与文档；不更改评分、人格、题目或体验 UI。生产上线后首先观察5—7天，避免同时进行产品实验。

**状态：** 预览测试已完成核心事件验收，待生产上线后核验。  
**分支：** `feature/smti-tracking-v1-20261015`  
**代码位置：** `index.html`。

## 1. 为什么埋点

现有 GA4 主要记录 `page_view`、`session_start` 等。SMTI 是单页面应用，答题、结果展示没有路由切换；因此单凭页面浏览无法知道有多少人开始、完成或分享。新版本把**真实的界面操作**映射为 GA4 自定义事件。

GA4 Measurement ID 已在 `index.html` 页头配置，本分支沿用已有 `gtag()`，不重复加载 SDK。

## 2. Tracking V1 事件字典

| Event | 触发点 / 代码 | 参数 | 口径与限制 |
|---|---|---|---|
| `test_start` | `startQuiz()` | `test_version` | 每次点击开始均可上报；重复测可能产生多次 |
| `answer_progress` | `pick()` | `answered_count`、`question_count`、`test_version` | 按**独立已回答题数**首次达到 1/4/7/11/14 时上报；改选不新增进度 |
| `test_complete` | `trySubmit()` 验证无漏答后 | `question_count`、`test_version` | 表示成功提交，不代表结果展示成功 |
| `result_view` | `doResult()` 调用 `renderResult` 与 `showScreen('result')` 后 | `test_version` | 表示页面切到结果态；不保证用户实际阅读 |
| `share_click` | `copyResult()` / `copyLink()` | `share_method`、`test_version` | 只是分享意图，不代表已分享 |
| `share_copy_success` | `navigator.clipboard.writeText()` 成功返回时 | `share_method`、`test_version` | 已复制到剪贴板；不代表朋友访问 |
| `share_landing` | 页面加载时检测带 `utm_source=smti_share` 且 `utm_medium=peer` 的 URL | `share_method`、`test_version` | 从本版专用分享链接进入的落地事件；同一用户可重复产生 |
| `retest_click` | `resetQuiz()` | `test_version` | 点击重新测一次，不等于再次完成 |

`share_method`：`result_text`（复制文字结果）或 `direct_link`（复制纯链接）；不发送具体结果、题目答案、姓名或用户 ID。

### 为什么不记录每一道“第 N 题完成”

现有14道题**同时展示**，用户可任意顺序答题。因此 `answered_count = 7` 是“累计答完7个独立题目”，**不等于第7题**，不能据此宣称第7题流失高。若将来要研究具体题目漏答，应进一步设计匿名、最低必要的题目未答率统计并评估隐私影响。

## 3. 归因链接

两个分享按钮现在分别生成：

```
https://www.smtitest.top/?utm_source=smti_share&utm_medium=peer&utm_campaign=smti_result&utm_content=result_text
https://www.smtitest.top/?utm_source=smti_share&utm_medium=peer&utm_campaign=smti_result&utm_content=direct_link
```

实际域名取运行时的 `window.location.origin`，可能有 `www` 与非 `www` 差别；GA4 应验证同一 Measurement ID 的会话来源。UTM 不传用户答案。不要把 `share_click` 当作实际分享成功，或把 `share_landing` 当作唯一新用户。

## 4. 第一阶段关注的指标

- **Start Rate** = 有 `test_start` 的用户数 / 有效登陆页用户数（需确认同一 GA4 身份口径）。
- **Test Completion Rate** = 有 `test_complete` 的用户数 / 有 `test_start` 的用户数。
- **Result Activation Rate** = 有 `result_view` 的用户数 / 有 `test_start` 的用户数。
- **Share Intent Rate** = 有 `share_click` 的用户数 / 有 `result_view` 的用户数。
- **Attributed Share Visits** = 使用受控 UTM 进入的会话 / 用户数；**不是**整体推荐归因。
- **Copy Success Rate** = `share_copy_success` 次数 / `share_click` 次数，含重复点击。

同一用户可能重测和刷新页面，用户计数、会话计数、事件计数不得混用。未建立匿名 test_attempt_id，V1 暂时无法精准连接每一次独立作答尝试。GA4 身份识别、同意状态、采样和丢事件均可能影响漏斗数值。

## 5. GA4 设置与校验

1. **不要先合并到 main。** 在 Railway → SMTI 服务 → Settings/Source/Deployments 核实部署来源分支、生产自动部署规则；测试分支不能误指向 Production。可使用独立 Preview 环境或本地 HTTP Server。
2. 在浏览器安装 Google Tag Assistant，或用 GA4 → Admin → DebugView 验证。`gtag` 能被执行不等于 GA4 服务端已收到。
3. 完成两次路径：
   - **路径 A：** 访问首页 → 开始 → 任意顺序答14题 → 改选某题 → 提交 → 等待结果 → 复制结果 → 返回首页。
   - **路径 B：** 用带 UTM 的链接打开 → 开始 → 答几题 → 关闭，不应记录完成。
4. 预期 A 每个 `test_start` 1次，`answer_progress` 对5个里程碑分别1次，`test_complete` 1次，`result_view` 1次；改选不新增进度。B 有 `share_landing` 和 `test_start`，没有 `test_complete`。
5. 在 GA4 自定义定义中创建**事件范围**自定义维度 `test_version`、`share_method`、`answered_count`（必要时 `question_count`）；自定义维度通常只对创建之后的事件生效，标准报表可能有延迟。
6. 确认 Google Tag 的使用告知、同意管理和实际用户隐私政策满足适用要求；不要上传敏感答案到分析系统。
7. 数据正式上线后先保持产品不变，收集5—7天作为基线；如果样本不足以分析明显差异，则延长观察。

## 6. 功能分支已做的静态 / 模拟测试

- 对 `index.html` 中浏览器 JavaScript 执行语法解析（PASS）。
- 模拟浏览器 DOM 和 gtag 调用，完成14题、改选、提交、结果、复制、重测（PASS）。
- 进度上报：1、4、7、11、14，每个一次（PASS）。
- 分享复制产生带 UTM 的 URL（PASS）。
- 事件参数未上传答案/人格名称（PASS）。

**尚未完成：** 真实浏览器全链路 QA、Railway Preview 构建、GA4 DebugView 实收事件、移动端及微信内浏览器兼容性。**这些检查完成前不合并。**

## 7. 另外发现的产品缺陷（本次不动，以免影响分析基线）

- `detectHidden()` 包含 `muscle` 的计数条件，但题目未发现 `ms` 标记。需单独验证该类型不可达。
- `server.js` 对任何请求都返回 `index.html`，若生产使用 Node Server 提供静态资源，本地 PNG 可能请求到 HTML；需在 Railway 部署环境核实。
- 角色图片失败回退、结果图章样式残留等属于后续独立修复，不混入跟踪功能分支。

## 8. 下一个开发决策点

先确认 Railway 的主生产分支和是否启用 PR/Branch Preview。确认后准备 Preview URL，测试 GA4 DebugView，再决定是否提交 main。上线后再根据真实漏斗选择 Activation 或 Distribution 实验，不在 Tracking V1 中提前做 A/B UI 改造。
