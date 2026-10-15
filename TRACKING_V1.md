# SMTI Growth Lab — Tracking V1 (待验收)

## 0. 环境隔离：Preview 默认为关闭（2026-10-15）

**代码已在本 PR 中隔离 GA4；正式站点的 main 未修改。** `index.html` 现使用主机名白名单：

- 生产：`www.smtitest.top`、`smtitest.top`、`smti-production.up.railway.app` → 保持原有 GA4 Measurement ID `G-GRM45XJCSH`。
- 其余主机名（Railway PR 域名、`localhost` 等） → 默认不加载 GA4 SDK、`smtiTrack()` 不上报事件，避免污染生产数据。
- **必须另外创建 GA4 测试 Property + Web 数据流**，获得新的 `G-...` Measurement ID。之后仅在功能分支中把 `previewMeasurementId = ''` 设置为测试 Property 的新 ID，重新检查 Preview 数据确实不进入生产 Property。**不要**填入原来的生产 ID。
- 本次只依赖 hostname 区分环境：如果预览域名未来变化，默认仍保持关闭；如果将真实生产域名用于预览，需先调整白名单，否则可能写入正式 Property。Railway PR 环境不能绑定生产的自定义域名。
- GA4 默认可将不同浏览器/设备识别为不同假名用户；预览测试也会有测试者的匿名数据，只应进入测试 Property。

**模拟验证：** 生产3个白名单主机只注入生产 GA4 脚本；模拟 `pr-1.up.railway.app`、`localhost`、`preview.smtitest.top` 等域名不注入 GA4 脚本（PASS）。仅为 JavaScript 执行模拟，尚未完成真实预览服务/GA4 实收事件验证。

**Railway 下一步（用户操作）：** Project Settings → Environments → Enable PR Environments；检查现有 Draft PR #1 是否生成独立 Preview。生产仍只关联 `main`；启用后可能产生额外费用。预览出现后检查网址、环境变量、是否与生产共享外部资源，确认再测试。


**状态：** 仅功能分支，未合并 main、未部署至生产。  
**分支：** `feature/smti-tracking-v1-20261015`  
**代码位置：** `index.html`（纯前端 HTML/CSS/JS）。  
**目标：** 先获得可靠的“进入 → 开始 → 答题 → 结果 → 分享”行为数据，再决定优化实验。**本次不改评分算法、题目与结果展示逻辑。**

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
