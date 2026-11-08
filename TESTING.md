# web-games 测试手册

> 给 AI 会话（和新同事）的完整测试方法。写这个文件的起因：2026-11-07~11-08 几轮修 bug 和加功能时，
> 「怎么测网页版、怎么测手机版」的方法只存在于当次对话里，新会话完全不知道，每次都要重新摸索。
> 现在全部落盘，动手测试前先读完这份。

## 0. 三分钟速查（最要命的几条）

1. **浏览器自动化的"真实鼠标键盘注入"在本环境不通**——不要反复尝试点击路径浪费时间，直接用页面内
   `evaluate()` 驱动（详见 §3.2）。这是实测过的结论，不是猜测。
2. **本地测试必须起 HTTP 服务**，浏览器打不开 `file://` 地址；测完记得关服务。
3. **开 PR 前必须核对 base 是 `main`**。叠分支的 PR 在下面的分支合并后**不会**自动改 base（分支没删除时），
   会出现「用户点了合并、改动却合进了分支、主干什么都没变」的静默失败（PR #22 实录，用户在手机上
   看不到新功能就是这么来的）。
4. **验证线上版本要加缓存穿透参数**（`?nocache=时间戳`）：GitHub Pages 的 CDN 缓存 10 分钟；
   手机浏览器同样有缓存，要下拉刷新。
5. 测试结论分层如实汇报：实测过什么、模拟过什么、没测过什么（例如无声浏览器验证不了真实发音音色）。

## 1. 仓库结构与发布链路

- 仓库位置：`C:\Users\liu\ZCodeProject\web-games`（GitHub: ly8427/web-games）。
- 每个游戏 = `games/<英文名>/index.html` 单文件，零外部依赖、离线可玩；
  新游戏要在根 `index.html` 的 `GAMES` 数组和 `README.md` 游戏列表里登记。
- 线上地址：https://ly8427.github.io/web-games/ —— 从 `main` 分支自动构建，合并 PR 后约 1 分钟生效。
- **Git 纪律**：Windows 侧本地分支提交；推送一律走 WSL：
  `wsl -e bash -c "cd /mnt/c/Users/liu/ZCodeProject/web-games && git push origin <分支>"`；
  一律开 PR，用户合并；禁止直接推主干。
- GitHub API（查 PR 状态、建 PR）：认证通路与 token 提取方法见本机 `~/.zcode/AGENTS.md`（不在公开仓库里复述凭据细节）。

## 2. 逻辑模拟测试（浏览器之前先做）

规则类修复（计分、距离、队列、解析）先用纯脚本模拟，再上浏览器：

```bash
node <<'EOF'
// 把页面里的核心规则抽成纯函数复刻，构造边界场景
// 例：双人模式"卡 90m"bug 的复现——5 词全对但答错过一次 = 90m，队列发尽 → 卡死
EOF
```

流程：**先复现坏结果（证明理解对了）→ 修复 → 再跑一遍证明好了**。
实例：双人模式答错词后输入框永久禁用的 bug（PR #20），就是先在 node 里复现"卡死在 90m"再修的。

## 3. 网页版实测（浏览器自动化）

### 3.1 准备

```bash
# 仓库根起服务（后台），另开命令确认
python -m http.server 8931 --bind 127.0.0.1     # 建议 run_in_background 方式起
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8931/games/word-runner/index.html   # 期望 200
```

测完关服务：`run_in_background` 起的用 TaskStop；手动起的
`netstat -ano | grep :8931` 找 PID 后 `taskkill //PID <pid> //F`。

浏览器走 **control-browser skill**（browser-use 插件）：
- 每次 `mcp__node_repl__js` 调用都是全新内核，**必须先跑 skill 的 bootstrap**（`setupBrowserRuntime`，
  插件根从 `process.env.ZCODE_PLUGIN_ROOT` 取）；
- backend 用 `await agent.browsers.getForUrl("http://127.0.0.1:8931/...")`；
- 动标签页前：**单独一次调用** `browser.tabs.list()` 看全量 → 下一次调用按 id `tabs.get(id)`；
  报 "tab unavailable" 就是 id 失效了，重新 list 拿新 id；
- 截图前先 `nodeRepl.write(await agent.documentation.get("screenshots"))`，
  截图必须同格 `nodeRepl.emitImage(await tab.screenshot())`。

### 3.2 输入注入的坑（本环境实证，2026-11-07）

三条"真实输入"通路**全部不可用**：

| 通路 | 表现 |
|---|---|
| playwright locator click（含 `force:true`） | 超时（可操作性等待卡住，元素能定位到） |
| `dom_cua.click({node_id})` | 返回成功但页面无反应 |
| `cua.click({x,y})` 坐标点击 | 无反应 |

**唯一可靠的驱动方式 = 页面内 evaluate**：

```js
// 点击
await pw.evaluate(() => document.getElementById('duelStartBtn').click());
// 填值 + 回车（游戏的 Enter 监听在输入框上，这样能真实触发处理函数）
await pw.evaluate((v) => {
  const i = document.getElementById('duelInput1');
  i.value = v;
  i.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', bubbles: true }));
}, 'apple');
```

如实汇报口径：这样驱动时，**游戏的处理函数、计时器、DOM 状态全部被真实执行和验证**；
没验证到的只有"操作系统级真实按键"这一层浏览器管道。

### 3.3 读状态、等异步、干净状态

- 读 DOM：`pw.evaluate(() => ({ zh: document.getElementById('dp1zh').textContent, ... }))`。
- **读页面内部变量**：顶层 `const`（`state`、`duel`、`WORDS`、`hintMode`）在 evaluate 里可直接访问
  （classic script 的顶层声明在同一 realm），例如 `state.current.en` 拿当前题的答案来驱动测试。
- 出题后的解锁有 450ms/1.5s/2.6s 计时器：用轮询等待
  `waitUntil(fn, ms)`（循环 `pw.evaluate(fn)` + `pw.waitForTimeout(120)`），别硬 sleep。
- 干净状态：`localStorage.clear()` 后 `tab.reload()`。
- `alert()`/`confirm()` 会挂住 evaluate：绕开会弹窗的路径；测删除流程前先
  `pw.evaluate(() => { window.confirm = () => true; })`。游戏的导入/保存反馈已改成页内消息，不弹窗。

### 3.4 单词跑酷测试锚点（元素 id 速查）

| 界面 | 锚点 |
|---|---|
| 主页 | `#menu`、难度卡 `.diff-card[data-diff=peace|easy|normal|hard]`、提示方式 `[data-hint=zh|audio]`、词库列表 `#bankList`（行内按钮 `[data-playbank]/[data-editbank]/[data-delbank]`）、`#newBankBtn` |
| 词库编辑 | `#bankNameInput` `#bankBulkInput` `#bankImportBtn` `#bankWordEn` `#bankWordZh` `#bankAddWordBtn` `#bankSaveBtn`，反馈消息 `#bankMsg` |
| 单人 | `#zhText` `#hintText` `#answerInput` `#feedback` `#speakBtn`，结算 `#result` `#resultTitle` `#nextBtn` |
| 双人 | 面板1 `#dp1zh` `#dp1hint` `#duelInput1` `#dp1fb` `#dp1dist` `#dp1speak`（面板2 把 1 换 2），设置页 `#duelStartBtn` `#duelCustomChips` `#duelSelMsg`，结算 `#duelResult` `#drTitle` `#drAgainBtn` `#drHomeBtn` |
| 页面内部状态 | `state.current.en`（单人当前题）、`duel.players[i]`（双人）、`hintMode` |

### 3.5 回归清单（改动词库/出题/判分后必跑）

1. 导入：正常行、**多词短句**（`air hostess 空姐`）、带连字符/撇号的词、缺中文的坏行 → 计数与切分结果；
2. 单人：短句提示行词距、连字符原样显示、答对答错反馈揭晓词义、通关进度保存；
3. 双人：**答错 1 次再答完全部词**（历史卡死 bug 的回归点，90m 处必须出现第 6 个词）、冲线结算、再来一局；
4. 听音拼写：出题界面无中文无首字母、重听按钮、答对揭晓；
5. 模式选择：切换、刷新后记住（localStorage `wg_hint_mode`）。

## 4. 手机版

### 4.1 本地视口模拟（改动后马上验）

```js
await tab.setViewportSize({ width: 390, height: 844 });   // iPhone 尺寸
await tab.reload();
// 检查目标区块：getBoundingClientRect() 顶部在视口内、宽度无溢出；配一张截图
```

历史上手机特有的坑：双击缩放吞点击（其他游戏，见 PR #17/#19）、长菜单需要滚动——
新加区块后用 390×844 过一遍最稳。

### 4.2 线上验证（合并后）

```js
// 1) Pages 构建状态（不带 token 会 404）
fetch("https://api.github.com/repos/ly8427/web-games/pages/builds/latest", {headers})
// status === 'built' 且 commit === main 最新提交
// 2) 线上内容验证（!nocache 穿透 CDN 的 10 分钟缓存）
fetch("https://ly8427.github.io/web-games/games/word-runner/index.html?nocache=" + Date.now())
  .then(r => r.text())
  .then(t => console.log(t.includes("新功能的特征字符串")));
```

手机端最后一步永远是：**让用户下拉刷新**（手机浏览器缓存最长 10 分钟）。

## 5. 汇报格式（对着用户说话时）

- 结论先行：修好没 / 卡在哪 / 下一步谁动；
- 「实测」「模拟」「推断」分层写，别混；
- 术语当场展开（合并请求、主干、缓存这些词对非工程读者要翻译）；
- 测试数据（localStorage）都在测试浏览器里，不影响用户设备。
