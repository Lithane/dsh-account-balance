# dsh-account-balance

DeepSeek Harness（DSH）侧栏底部的**账户余额与花费卡片**：常驻显示账户余额和今日花费，
点击展开五层级成本明细，带峰谷时段状态与一键充值入口。

> 本项目是 [LL-cmyk-so/dsh-balance-widget](https://github.com/LL-cmyk-so/dsh-balance-widget)
> 的修改版，自 0.1.0 起独立演进。上游完整文档保留在 [docs/upstream/](docs/upstream/README.md)。

## 功能

- **账户余额**：优先读 DSH 内置账号服务 `deepseekAccount`（与官方「设置 → 账号与余额」同源，
  **不需要 API Key**）；未登录或旧版宿主时回退到官方 `GET /user/balance` + 已存的 `DEEPSEEK_API_KEY`
- **成本估算**：最近一次提问 / 今日·本会话 / 今日·本工作区 / 今日·所有工作区
- **峰谷状态**：按北京时间工作日 09:00–12:00、14:00–18:00 判峰，含中国法定节假日表
- **官方价格自动同步**：启动时 + 每 12 小时抓取官方定价页，失败回退内置价表
- **模型工具**：`deepseek_billing`，可直接问模型「余额多少 / 今天花了多少」
- 卡片每 60 秒自动刷新；点开弹层立即强刷

## 安装

桌面端（DSH Desktop）：**设置 → 插件 → 添加插件**，填写包名或仓库地址，然后**重启应用 + 硬刷新**。

```sh
# 网页端
dsh plugin --profile web add @lithane/dsh-account-balance
# 或直接装仓库
dsh plugin --profile web add github:Lithane/dsh-account-balance
# 或本地开发目录（改动即生效，最适合开发）
dsh plugin --profile web add link:/绝对路径/dsh-account-balance
```

## 配置

写在 profile 的 `cordis.patch.yml`（桌面端是 `~/.dsh/profiles/desktop/`）：

```yaml
- id: account-balance
  name: '@lithane/dsh-account-balance'
  config:
    balanceBaseURL: https://api.deepseek.com   # 回退路径的余额接口
    balanceApiKeyEnv: DEEPSEEK_API_KEY         # 凭据服务里的 ref
    requestTimeoutMs: 5000
    modelId: deepseek-v4-flash                 # 计价模型
    lowThreshold: 5                            # 余额低于此值变黄
    criticalThreshold: 1                       # 余额低于此值变红
```

更多配置项与定价说明见 [docs/upstream/README.md](docs/upstream/README.md)。

## 架构

**没有构建步骤**：`lib/` 就是源码，改完直接生效（上游也是这个结构）。

```
lib/index.js          宿主半（Node）
  ├ 5 条 HTTP 路由      /api/account-balance/{balance,active-cost,today-cost,last-cost,cost}
  ├ 会话日志解析        遍历 ~/.dsh/sessions/ 算 token 与费用
  ├ 余额读取            账号服务 deepseekAccount → 回退凭据服务 + /user/balance
  └ 模型工具            deepseek_billing
lib/client.js         浏览器半（React）
  ├ 注入槽位            ctx.slots.inject("sidebar.footer.action")
  ├ 自注入 CSS          <style data-plugin="account-balance" data-plugin-css="account-balance/styles.css">
  └ 卡片 + 弹层         弹层 portal 到 document.body，用 ResizeObserver 跟随侧栏宽度
cordis.patch.yml      把插件行插入 profile
package.json          的 dsh 字段：bundle.patch + client.platform = "web"
```

### 几个必须知道的机制

| 主题 | 要点 |
|---|---|
| 会话日志 | `~/.dsh/sessions/<工作区编码>/session-<id>/session.vN.jsonl.zstd`。当前是 **v4**，append-only，**每次 flush 追加一个独立 zstd 帧**——必须按帧解压（Node 一次性解压只返回第一帧，会静默丢掉 99% 数据）。`zlib.zstdDecompressSync` 需要 **Node ≥ 22.15** |
| 路径解析 | `sessionsRoot()` 按 `DSH_HOME` → `USERPROFILE` → `HOME` 回退。**Windows 不设 `HOME`**，且 `??` 对空字符串不回退，所以别写 `${process.env.HOME ?? ""}` |
| 样式去重 | CSS 按 `data-plugin-css` 的 tagId 去重。**改 tagId 要同步改**，否则与其它版本共存时你的样式会被判定"已注入"而静默跳过 |
| 样式类名 | 全部走 `dshbw_` 前缀（沿用上游）。颜色/圆角优先用 `--dsw-*` 设计 token，自动跟随明暗主题 |
| 金额排版 | `moneyNodes()` 用正则按第一个数字把金额拆成「符号 / 整数段 / 小数段」，分别套 `.dshbw_sym`/`.dshbw_int`/`.dshbw_dec`；`line-height:1` 保证不影响卡片高度 |
| 卡片标签 | `.dshbw_brand`（全称）与 `.dshbw_brandMini`（侧栏收起成 36px 图标时显示），靠 `[data-compact="true"]` 切换 |
| 与其它卡片共存 | `sidebar.footer.action` 是**不换行**的 flex 行，多张"占满一行"的卡片会被压成半行。**注意卡片与那一行之间还有一层 DSH 槽位出口容器，它是 `display: contents`（不生成盒子），却正是 `el.parentElement`——往它上面写 `flex-wrap` 完全无效**，必须从卡片向上爬到最近的 computed `display: flex` 祖先再写。本卡片根元素为 `flex: 1 1 100%`；换行在**那一行元素上做引用计数**（属性名 `__dshBalanceCardWrap`，是与 Liaobots 插件约定的跨插件契约）：首个声明者记录原值并打开换行，最后一个离开的还原。因此任意数量的卡片共用一行都不抢，卸载顺序也不会压扁仍在挂载的邻居。改这块务必同时守住"爬到 flex 祖先"和这个契约 |

## 开发

```powershell
# 语法门（无构建步骤，改完直接查）
npm run verify          # = node --check lib/index.js && node --check lib/client.js

# 本地安装调试
dsh plugin --profile web add link:D:\path\to\dsh-account-balance
```

**改动生效方式不同，别搞混**：

- **客户端半（`lib/client.js`）**：硬刷新页面通常即可
- **宿主半（`lib/index.js`）**：**必须重启 DSH**（宿主代码只在启动时加载）

**三个接口可直接自测**（端口每次启动随机，用当前 GUI 的地址）：

```powershell
(Invoke-WebRequest 'http://127.0.0.1:<port>/api/account-balance/balance'     -UseBasicParsing).Content
(Invoke-WebRequest 'http://127.0.0.1:<port>/api/account-balance/active-cost' -UseBasicParsing).Content
(Invoke-WebRequest 'http://127.0.0.1:<port>/api/account-balance/today-cost'  -UseBasicParsing).Content
```

`active-cost` 返回 `no session files found` = 会话路径解析有问题（见上表「路径解析」）。

### 调试经验

- 样式不生效时，先查 `<style data-plugin-css="account-balance/styles.css">` 是否在 `document.head` 里
- **卡片整个消失但日志没有报错**：多半是渲染时抛了异常——槽位对崩溃条目的处理是**整条摘掉**
- 改 `.js` 里的 CSS 字符串时注意：它是**单行 JS 字符串**，双引号要写成 `\"`

## 发布

```powershell
# 1) 改 package.json 的 version，并往 CHANGELOG.md 加一节
# 2) 只做本地分发（不需要 npm 账号）
npm pack                                            # 产出 .tgz，别的机器 dsh plugin add <路径>
# 3) 发布到 npm（别的机器可按包名直接安装）
npm login
npm publish --access public                         # scoped 包必须带 --access public
```

- **同一版本号不能重复发布**，每次改动都要 bump
- 发版后建议 `git tag v<version> && git push --tags`
- ⚠️ **DSH 有 peer 兼容闸门**：启动时比对 `peerDependencies` 里 `@deepseek-ai/dsh` 的范围
  （**预发布版本参与匹配**）。不匹配且未豁免的 bundle 会被**静默跳过**（只记进 `skippedBundles`，不报错）。
  改这个范围时要确认覆盖目标 DSH 版本，否则用户升级 DSH 后插件会凭空消失。
  DSH 0.2.0 起还新增了这道闸门（上游 0.6.1 就是为此把范围从 `<0.2.0` 放宽到 `<0.3.0`）。

## 已知限制

- **位置固定在 `sidebar.footer.action`**（侧栏底部、设置上方），改不了；想换位置只能改槽位注册
- **macOS 桌面端收起侧边栏时整个左栏被隐藏**，卡片会一起消失，而不是变成 36px 图标——
  这是 DSH 的布局行为（`data-platform='darwin'` 下不留图标轨），非本插件问题
- 成本数字为**本地估算**，不是官方账单；`web_search` 的辅助调用无 usage 字段，按标定常数估算（±30%）
- 会话解压依赖 Node 内置 zstd，需 **Node ≥ 22.15**
- 价格表与中国法定节假日表需跟随官方更新（节假日表每年 11 月左右发布次年安排）

## 上游与许可

上游 [LL-cmyk-so/dsh-balance-widget](https://github.com/LL-cmyk-so/dsh-balance-widget)（MIT），
本仓库基于其 **0.6.3**。原始版权归原作者，[LICENSE](LICENSE) 保留上游版权行并追加本 fork 的版权行；
上游完整文档（含配置项、定价说明、历史版本）见 [docs/upstream/](docs/upstream/README.md)。

两版可共存（路由与全部标识已改名），但会同时往同一个侧栏槽位注入卡片，视觉上重复，不建议同时启用。

## License

MIT，继承上游。
