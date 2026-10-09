# 变更日志

本文件记录 **dsh-account-balance** 自身的改动。
上游 dsh-balance-widget 的历史见 [docs/upstream/README.md](docs/upstream/README.md) 的「版本历史」一节。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [0.1.2] - 2026-10-09

### 修复

- **与上游 `dsh-balance-widget` 同时启用时直接报错**。模型工具名沿用了上游的
  `deepseek_billing` 而没有改名，而 DSH 在组装工具声明时对重名是**硬报错**
  （`Duplicate tool declaration name: deepseek_billing`），所以 0.1.1 说的
  "两版可共存"并不成立。现改名为 **`account_balance`**，两版并存不再冲突
  （剩下的只是同一个槽位里两张外观相同的卡片）。
- **从 GitHub 安装后卡片不出现**。客户端模块 id 写的是 `account-balance`，而包名是
  `@lithane/dsh-account-balance`。DSH 的客户端模块表按**包名**索引——实测 harness 自带的
  68 个客户端模块、以及能正常工作的 `dsh-balance-widget`、`dsh-liaobots-balance`，
  其 `window.__ModuleLoader__.load({ id })` 全部等于各自包名。id 与包名不一致时
  bundle 能被加载，但槽位不会挂载，卡片**静默消失**（不报错、日志也看不到）。
  现把 id 对齐为包名。

## [0.1.1] - 2026-10-09

### 变更

- **与其它侧栏卡片协作换行**。`sidebar.footer.action` 是一个**不换行**的 flex 行，
  两张都想占满一行的卡片会被各自压成半行（表现为挤在一起而不是上下叠放）。
  现在本卡片自己保证该行可换行：

  - `flex` 由 `1 1 auto` 改为 `1 1 100%`，明确声明"占满一行"；收起态
    （`[data-compact="true"]` 的 `flex: 0 0 auto`）优先级更高，不受影响。
  - ⚠️ **换行必须写到真正的 flex 行上**。卡片与那一行之间还有一层 DSH 槽位出口容器，
    它是 `display: contents`——**不生成盒子**，却正是 `el.parentElement`，所以往它上面写
    `flex-wrap` 完全无效（本插件与 Liaobots 插件最初都踩了这个坑，表现为两张卡并排各占半行）。
    正确做法是从卡片向上爬到最近的 `getComputedStyle(x).display === "flex"` 祖先。
  - 换行在那一行元素上做**引用计数**（属性名 `__dshBalanceCardWrap`，是与 Liaobots 插件约定的
    跨插件契约）：首个声明者记录 `style.flexWrap` 原值并打开换行，最后一个离开的还原。
    这样任意数量的卡片共用一行都不抢，**任意卸载顺序**也不会压扁仍在挂载的邻居。
  - 中途走过的弯路（已修）：曾用"行内已无其它子元素"作为释放条件，但那一行只挂着一个
    `display: contents` 出口容器，`childElementCount` 恒为 1，导致**单独一张卡也永不还原**
    （Liaobots 插件的两项前端测试因此失败，改为引用计数后 18/18 通过）。

  这样无论两张卡片的挂载/卸载顺序如何，都不会互相破坏。

## [0.1.0] - 2026-10-05

首个版本：基于上游 `dsh-balance-widget@0.6.3` 的修改版，自此独立演进。

### 修复

- **Windows 上读不到会话文件**（上游 bug）。`lib/index.js` 的 `sessionsRoot()` 原为
  `${process.env.HOME ?? ""}/.dsh/sessions`：Windows 不设 `HOME`（值为**空字符串**而非 `undefined`），
  而 `??` 只对 `null`/`undefined` 生效，路径被拼成 `/.dsh/sessions`；读取失败又被 `catch {}` 静默吞掉。
  表现为**余额正常但用量/费用全空**，三个接口返回 `no session files found`。
  现改为 `DSH_HOME` → `USERPROFILE` → `HOME` 回退链，并把反斜杠归一为 `/`。

### 变更

- **金额排版分层**：金额原本拼成单一字符串整体渲染，整数位与两位小数同字号、不便速读。
  现拆成「货币符号 + 整数段（`1.3em`）+ 小数段（`0.8em`）」三段。用正则按第一个数字切分，
  兼容 `¥55.06`、`¥0.0133`、`¥1,234.5678`、`¥55`（无小数）、`USD 12.5` 等形态，
  `—`/`…` 原样透传；`line-height:1` 保证卡片高度不变。
- **卡片图标改为文字标签**：左侧原为钱包形状 SVG（`rect` + 一条横线，易被误认为日历图标，
  与其它插件难以区分）。现改为文字 **DS账户余额**；侧栏收起成 36px 图标时显示短标记 **DS**。

### 内部

- 全量改名以支持独立演进：包名 `@<user>/dsh-account-balance`；
  行 id、`export const name`、模块 id、i18n 命名空间 `NS`、CSS tagId、日志前缀统一为 `account-balance`。
- HTTP 路由前缀由 `/api/dsh-balance` 改为 `/api/account-balance`，因此可与上游版本共存。
- 文档重组：本项目的 README 独立编写，上游文档归入 `docs/upstream/`。
