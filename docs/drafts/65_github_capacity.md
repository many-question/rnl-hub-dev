# GitHub 承载评估：统一 IP 设计库

- 日期：2026-11-04
- 状态：讨论稿 v0.3，尚未实施。
  - v0.2 按人类研究员当天的 5 条澄清修订（第 1 节）。
  - v0.3 记下已定事项（第 7 节），并把网站改为现场抓取网表（第 5 节）。
  - v0.1 推荐的"拉取式集成"是为私有仓库设计的，平台确定公开后不再需要。
- 问题：
  - 把所有东西都看成 IP，平台就成了一个多 agent 参与的 IC 设计库。GitHub 撑不撑得住；
  - 公开平台要不要建组织、要不要付费；
  - 网站上用 RNL 查看器看设计，会不会碰到下载或并发限制。
- 关联：[64 号文档](64_multi_agent_collaboration.md)。64 号文档从 v0.3 起吸收了本文的结论（v0.4 把 IP 目录改为 `design/<IP>/`）；本文第 2 节里的"64 号文档 §14"指 v0.2 的编号，在 v0.3 里对应第 15 节
- 数据来源：
  - GitHub 官方文档；
  - 2026-11-03/04 用 `gh api` 和 `curl` 查询了本仓库、ADC-12b-100M、NixOS/nixpkgs、rich-netlist 站点和两个账号。

## 0. 结论先行

- **组织**：建一个公开组织，Free 套餐就够，现在不需要付费。
  - Free 组织的公开仓库已经有：分支保护、规则集、CODEOWNERS 审批、合并队列、自动合并、免费的 Actions 用时和 Pages。
  - 以后最可能碰到的是网站带宽。Pages 的用量限制在文档里没有按套餐区分，所以网站扩容要靠 CDN，升级套餐不解决问题。
- **仓库容量**：贡献只有网表和有大小上限的关键证据。
  - 典型用量下仓库每年增长约 0.2–0.3 GB；全部顶格也只有每年 2–3.4 GB。
  - GitHub 建议 .git 不超过 10 GB，按顶格估计也有三到五年。
- **网站和查看器**：
  - 站点只放查看器和索引，网表和证据文件在浏览时现场抓取。站点因此很小，几 MB 到几十 MB，碰不到 Pages 的 1 GB。
  - 现场抓取用 jsDelivr，按提交号读取 GitHub 上的文件，`raw.githubusercontent.com` 作备用。GitHub API 匿名每小时只有 60 次，不能用来浏览。
  - GitHub Pages 由 Fastly CDN 分发，官方没有公布并发上限。查看器首次打开约 1.3 MB（gzip 后），每月 100 GB 的带宽软限大约够 7–8 万次首次打开；回访走缓存。
  - 有些信息不能现场抓取：可信度、证据等级、预算是否满足、排行。这些派生信息要在建站时算好，放进索引。
- **公开带来的后果**：网表里没有 PDK 数据。晶体管级的复核只有持有同一工艺 PDK 和相应仿真器的参与者能做；模型级复核的门槛取决于统一 tb 用哪种仿真器。

## 1. 2026-11-04 的澄清及其影响

| 澄清 | 对设计的影响 |
|---|---|
| 1. 这是公开平台 | 公开组织加 PR 流程（第 3 节）；v0.1 的 A/B/C 三条路不必再选 |
| 2. 平台上主要是 RNL 网表，不带 PDK 数据，只写工艺名；参与者在内部做设计和仿真，只推送网表 | CI 检查网表里没有 PDK 内容（3.5）；统一 tb 只引用工艺名和标准角名称，各参与者在本地映射到自己的 PDK |
| 3. 只接收验证指标结论的关键证据，每个验证贡献有大小上限 | 仓库容量不再是问题（第 4 节）；不存图片和波形，网站上的图由数据现画 |
| 4. 每个 IP 的定义唯一（含工艺、接口、指标）；指标数值可以有多套（目标、模型、各实现版本的数据）；实现可以有多组版本树；引用其他 IP 必须写到具体版本 | 第 2 节 |
| 5. 网站上用 RNL 查看器直接看设计 | 第 5 节 |

## 2. IP 结构

### 2.1 组成

- **定义（每个 IP 唯一）**：功能、工艺、端口、指标清单。指标清单只写名字、条件和测量方法（tb），不写数值。
- **数值集**：同一份指标清单可以挂多套数值，每套都是追加的记录。
  - 目标：定义者给的目标，以及每个引用它的组合实现给的预算；
  - 模型：VA 行为模型的参数值；
  - 实测：各实现版本在 tb 上的结果，带证据等级和可信度。
- **实现**：可以有多组，每组一棵版本树。
  - 模型：VA 行为模型。定义者提交定义时必须同时提交第一版（64 号文档的规则 4）；
  - 组合：子 IP 实例、连线、每个实例的预算；
  - 电路：晶体管级 RNL 网表；
  - 数字模块的形式仍待定（64 号文档 §14 第 4 项）。
- **根 IP** 就是 mega-precision。它的指标清单就是统一 tb 和指标，目标数值由人类给出；它的实现就是以前说的"方案"。

网站给每个 IP 一张数据表：行是指标，列是各套目标、模型和各实现版本的实测值。预算满足与否一眼可见。

### 2.2 引用一律写到具体版本

- 地址格式建议用 `<IP>@<实现组>/v<n>`，例如 `seg-dac@r08-seg/v2`、`seg-dac@model/v1`。定义是唯一的，直接用 `<IP>` 引用。
- 组合实现里，每个实例都固定到一个具体版本。子 IP 还没有电路实现时，就固定到它的模型版本。
- 仿真结果的记录里，可以把某个实例换成另一个具体版本。例如把 `det` 从 `err-detect@model/v1` 换成 `err-detect@r05-het/v1`。
  - 这仍然是"写到具体版本"，同时不必为每种搭配都出新版本或分叉。
  - 证据等级按实际用到的版本计算：模型级、混合级或晶体管级。

```yaml
# design/mega-precision/impl/r07-loop/v2/impl.yaml
kind: composition
instances:
  dac: {ip: seg-dac,    use: r08-seg/v2, budget: {inl: 0.5, noise_density: 8}}
  det: {ip: err-detect, use: model/v1,   budget: {spur_floor: -150}}
```

### 2.3 已定的两点

人类研究员于 2026-11-04 同意以下两点：

1. **定义可以追加指标。**
   - 端口和工艺永不改变；
   - 指标清单只许定义者追加新指标，不许改已有指标；
   - 要改已有内容就是新 IP，用 `based_on` 记下来源。
2. **结果可以替换实例版本（2.2 第 3 条）。** 替换上去的也必须是具体版本。

另外，"定义唯一"防不住两个人用不同名字定义同一个功能。网站可以按接口指纹（64 号文档）提示疑似重复，任何人都可以追加一条"等同"记录。

## 3. 公开组织与套餐

### 3.1 为什么建组织

- 合并队列只对组织的公开仓库开放（另有 Enterprise Cloud 的私有仓库）。个人账号的仓库用不了。
- CODEOWNERS 可以指定团队，例如规格维护者，不必写死个人账号。
- 组织可以有多个所有者，平台不依赖某一个人的账号。
- 可以安装组织级的 GitHub App，合并机器人要用（3.3）。
- 名字中立：外部参与者加入的是平台，而不是某个人的账号。

建议新建一个专用组织，名字面向整个设计库，而不只是 mega-precision，因为 mega-precision 只是库里的第一个根 IP。现有的 `mega-precision-report` 继续放旧 run 的私有存档。

### 3.2 Free 够不够用

- 够用。理由见第 0 节：公开仓库需要的功能 Free 组织都有，Pages 的限制也不随套餐变化。
- 以后可能需要付费的情况：
  - Actions 并发不够。Free 同时跑 20 个任务，Team 60 个。格式检查一次一两分钟，几十个 agent 用不满 20 个；
  - 需要受保护的私有仓库，例如提交前的私有暂存区。目前没有这个需求，参与者可以在自己的 fork 或私有仓库里准备。

### 3.3 公开仓库上的提交流程

- 参与者用自己的 GitHub 账号 fork 主仓库，然后提 PR。
- 来自 fork 的 PR 运行工作流时拿不到仓库机密。格式检查本来也不需要机密。
- GitHub 默认要求所有首次贡献者的工作流先经维护者批准，这和"人只观察"冲突。可选的设置有三档：
  - 只对新注册 GitHub 的首次贡献者；
  - 所有首次贡献者；
  - 所有外部贡献者。

  建议用第一档。
- 合并由一个以 GitHub App 身份运行的机器人完成。合并条件：
  - 必需检查全部通过；
  - 改动只落在贡献目录里；
  - 提交者和记录里写的作者一致。
- 机器人不检出 PR 里的代码。它用 App 身份而不用 `GITHUB_TOKEN`，因为 `GITHUB_TOKEN` 做的合并不会触发后续的建站工作流。
- 受保护路径（spec、章程、工具、`.github/`）要 CODEOWNERS 批准，机器人不合并这些路径的改动。
- 合并后自动删除分支。

### 3.4 身份

- 每个参与者用自己的账号。GitHub 条款规定每人最多一个免费机器账号，所以一个人的多个 agent 通常共用一个机器账号，agent 名写在贡献记录里。
- 复核的独立性按账号（即操作者）计算更稳妥，网站同时显示 agent 和模型。
- 由此带来的后果：R01–R10 如果都用同一个机器账号提交，它们之间的互相复核只算同一个操作者。这一点需要人类研究员决定。

### 3.5 "不带 PDK 数据"由 CI 检查

- 网表里不得出现：
  - 指向贡献目录以外的 include 或 lib 语句；
  - 器件模型卡；
  - 加密段；
  - 绝对路径。
- 统一 tb 只引用工艺名和标准角名称（如 `tt`、`ss`），通过一个工艺绑定文件加载模型。这个文件由各参与者在本地准备，不提交。
- 器件名和尺寸会留在网表里，这是网表本身的内容。能否公开以各工艺的保密协议为准。公开的 git 历史被 fork 或缓存后无法收回。

## 4. 仓库容量

### 4.1 建议的上限

- 每个验证贡献总共不超过 256 KB，只收数据文件（YAML、CSV、JSON），不收图片和波形。
- 每个网表文件不超过 1 MB（GitHub 建议的单文件上限），每个设计贡献总共不超过 2 MB。
  - 每个实现只放本层网表，子 IP 只引用版本、不内联，所以文件天然很小。
  - ADC 现有 31 个 Spectre 网表，合计约 0.4 MB。
- 上限写进统一规则，由 CI 检查，超限的 PR 不能合并。

### 4.2 增长估计

假设每月有 1,000 个验证贡献、300 个设计版本：

| 情形 | 每月新增（未压缩） | 每年仓库增长（git 压缩 3–5 倍后） |
|---|---|---|
| 典型：验证平均 50 KB，设计平均 100 KB | 约 80 MB | 约 0.2–0.3 GB |
| 全部顶格：验证 256 KB，设计 2 MB | 约 860 MB | 约 2–3.4 GB |

- 即使全部顶格，也要三到五年才到 GitHub 建议的 10 GB。
- 对照：ADC 不设上限，8 天长到约 450 MB，其中 98% 是 PNG 和 JSON 结果文件（v0.1 第 2 节）。

### 4.3 其他 GitHub 限制

| 项目 | GitHub 的数（性质） | 本平台 | 判断 |
|---|---|---|---|
| 目录宽度 | 3,000 项（建议） | 版本和结果目录会持续增加 | 按 IP、实现组和月份分层 |
| 推送 | 每仓库每分钟 6 次（建议） | 参与者推送到自己的 fork，主仓库只接收合并 | 没问题 |
| 合并 | 每分钟 1 个 PR（建议） | 每天约 100 个 | 没问题；需要时开合并队列 |
| 未关闭 PR | 同一目标分支 1,000 个（建议） | 几十到几百个 | 要清理过期草稿 |
| 内容创建 | 每个身份每分钟 80 次、每小时 500 次 | 按各参与者自己的账号计；合并机器人每天几百次 | 没问题 |
| Actions | 公开仓库免费；Free 组织同时 20 个任务 | 格式检查一次一两分钟 | 没问题 |

## 5. 网站与 RNL 查看器

### 5.1 实测

- **Pages 由 CDN 分发。** 响应头带 `Via: 1.1 varnish`、`X-Served-By: cache-…`、`X-Fastly-Request-ID`，说明由 Fastly CDN 分发。静态文件的 `Cache-Control` 是 `max-age=600`。
- **GitHub 公布的 Pages 限制只有这几条：**
  - 站点不超过 1 GB；
  - 部署不超过 10 分钟；
  - 每月 100 GB 带宽（软限）；
  - 每小时 10 次构建（软限，用自己的 Actions 工作流发布时不适用）；
  - "可能限速"，返回 429。

  没有公布并发上限。
- **查看器的下载量。** RNL 网页查看器在 `rich-netlist.github.io/rnl/viewer/`。页面引用的全部文件未压缩 3.9 MB，gzip 传输 1.24 MB。其中 WASM 运行时 2.74 MB，gzip 后 0.88 MB。
- **ADC 的做法。**
  - 网站链接到托管的查看器，参数里传入 ADC 自己站点上的清单。
  - 清单固定到某个提交，列出每个文件的 sha256 和大小。
  - 网表复制在 ADC 自己的 Pages 站点下（`/repo/`），不经过 `raw.githubusercontent.com`。
- **现场抓取的三个入口。** 2026-11-04 用 ADC 的一个网表做了匿名请求实测：

| 入口 | 结果 | 缓存 | 限制 |
|---|---|---|---|
| GitHub API（contents） | 可用 | — | 匿名每个 IP 每小时 60 次 |
| `raw.githubusercontent.com` | 可用，允许跨域 | 5 分钟 | GitHub 2025-05-08 宣布收紧匿名下载限速，没有公布数字；社区有 429 报告 |
| jsDelivr `/gh/`，固定提交号 | 可用（ADC 仓库约 450 MB 也能取），允许跨域 | 一年，immutable | 单个文件超过 20 MB 默认不支持；第三方服务，没有 SLA |

### 5.2 站点里放什么

v0.2 照 ADC 的做法把网表和证据文件复制进站点，所以估计站点约一年就会超过 1 GB。改为现场抓取后，站点只有三样东西：

| 内容 | 大小 | 说明 |
|---|---|---|
| 固定版本的查看器 | 约 3.9 MB（gzip 传输 1.24 MB） | 不随设计数量增长 |
| 索引 | 几 MB 到几十 MB | 对象列表、引用关系、数值集，以及建站时算好的派生信息；按 IP 分片，打开哪个 IP 才加载哪一片 |
| 页面代码 | 很小 | |

- **派生信息在建站时算。** 可信度、证据等级、预算是否满足、排行、看板，都要在建站时算好。如果在浏览器里现算，就得先读完全部记录，可能是成千上万个文件。
- **每个对象记下合并时的提交号。** 对象只增不改，所以这个提交号下的文件永远有效，可以长期缓存。
- **打开设计时：**
  - 查看器清单（manifest）的 `base` 指向 jsDelivr 上该提交号的地址。ADC 的清单已经有 `base` 字段。
  - 证据文件（CSV 等）等用户点开图表时再抓取。

### 5.3 建议

- **抓取入口用 jsDelivr，`raw.githubusercontent.com` 作备用。** 入口写在建站配置里，出问题时整站一起切换。不用 GitHub API。
  - jsDelivr 上按提交号的文件缓存一年，不受 GitHub 匿名限速影响；
  - 它是第三方服务，没有 SLA，所以要保留切回 raw 的能力。
- **查看器固定版本，放在平台自己的站点上。** 好处有两点：
  - 查看器的流量不和 RNL 官网及其他项目共用 `rich-netlist/rnl` 的 100 GB；
  - 查看器升级不会意外改变平台页面。10-26 到 10-30 之间它发布了三个版本。

  rnl 的 Release 目前只有桌面安装包，有两种做法：
  - 在 CI 里按固定标签从 `apps/web` 构建；
  - 给 rnl 的 Release 加一个网页版压缩包。这是 rich-netlist 那边的改动。

  如果网页版发布到 npm，jsDelivr 也能分发查看器，Pages 的流量就几乎只剩索引了。
- **容量。** Pages 的流量主要是查看器：100 GB/月约合 7–8 万次首次打开，回访走缓存。网表和证据的流量落在 jsDelivr 或 GitHub 上。
- **代价：**
  - 网站要依赖 GitHub 或 jsDelivr 在线；
  - 首次抓取比同源加载慢一些：实测 raw 0.8 秒，jsDelivr 缓存未命中时 2.0 秒。之后都走缓存。

## 6. 规模再大时

- 数百个 agent，或者多个芯片项目共用一个库：
  - 按项目或按年分仓，用一个总索引连起来；
  - 读路径换成 CDN 上的索引。先例是 CocoaPods、Homebrew 和 Cargo：用户多了以后，三者都把"所有客户端克隆 GitHub 上的 git 仓库"改成了 HTTP 或 CDN 读取。
- 版图（GDS）进入范围后必须出库。

## 7. 待定

2026-11-04 已定：
- 新建一个 Free 公开组织；
- 4.1 的大小上限；
- 2.3 的两点；
- 复核独立性按账号计算；
- 网站现场抓取网表（第 5 节）。

仍待定：

1. 组织名称和所有者。
2. R01–R10 用什么账号提交（3.4）。
3. 统一 tb 用哪种仿真器。这决定谁能做模型级复核。
4. 讨论放在 GitHub issue 还是仓库文件里。
   - 公开后参与者都有自己的账号，issue 的 API 限额按各自账号计算，不再是瓶颈。
   - 放在仓库文件里的好处是网站能直接读，也不绑定 GitHub。

## 附：v0.1 的实测参照

| | mega-precision（现在） | ADC-12b-100M | NixOS/nixpkgs |
|---|---|---|---|
| 时间 | 2026-08-18 起 | 2026-10-26 至 11-02，8 天 | 2012 年起 |
| 仓库体积 | 约 7 MB | 约 450 MB | 约 3.5 GB |
| 当前文件 | 205 个 | 2,187 个，共 167 MB（PNG 118.7 MB，JSON 46.2 MB，其余约 2.5 MB） | — |
| PR | — | 84 个，合并 75 个，最多一天 19 个 | 51.7 万个，未关闭 1.27 万个；2026-10-01 以来约 380 个/天 |
| 所有者与可见性 | 个人账号，私有 | 个人账号，公开，main 未设保护 | 组织，公开 |

`many-question` 是 Free 个人账号，`mega-precision-report` 是 Free 组织。两者的私有仓库查询分支保护都返回 403："Upgrade to GitHub Pro or make this repository public to enable this feature"。

## 来源

- GitHub Docs：
  - [Repository limits](https://docs.github.com/en/repositories/creating-and-managing-repositories/repository-limits)
  - [About large files on GitHub](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github)
  - [Rate limits for the REST API](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
  - [Actions limits](https://docs.github.com/en/actions/reference/limits)
  - [GitHub Pages limits](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits)
  - [About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
  - [Managing GitHub Actions settings for a repository](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)（fork PR 的审批选项）
  - [Registering a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)
  - [Git LFS billing](https://docs.github.com/en/billing/concepts/product-billing/git-lfs)
- [GitHub Terms of Service](https://docs.github.com/en/site-policy/github-terms/github-terms-of-service)（账号条款）
- GitHub Changelog：
  - [Merge queue GA（2023-07-12）](https://github.blog/changelog/2023-07-12-pull-request-merge-queue-is-now-generally-available/)
  - [Updated rate limits for unauthenticated requests（2025-05-08）](https://github.blog/changelog/2025-05-08-updated-rate-limits-for-unauthenticated-requests/)
- GitHub Community 上的 429 报告：[#159123](https://github.com/orgs/community/discussions/159123)、[#177971](https://github.com/orgs/community/discussions/177971)
- [2026 pricing changes for GitHub Actions](https://github.com/resources/insights/2026-pricing-changes-for-github-actions)
- [Cloudflare Pages limits](https://developers.cloudflare.com/pages/platform/limits/)
- [jsDelivr README](https://github.com/jsdelivr/jsdelivr)（GitHub 文件的大小限制和缓存）
- 先例：
  - [CocoaPods#4989](https://github.com/CocoaPods/CocoaPods/issues/4989)
  - [Homebrew 4.0.0](https://brew.sh/2023/02/16/homebrew-4.0.0/)
  - [Rust 1.68.0](https://blog.rust-lang.org/2023/03/09/Rust-1.68.0/)
  - [nixpkgs RFC 140](https://github.com/NixOS/rfcs/blob/c8569f6719356009204133cd00d92010889ed56d/rfcs/0140-simple-package-paths.md)
- 实测：
  - `rich-netlist.github.io/rnl/viewer/` 的文件大小和响应头；
  - ADC 的清单 `many-question.github.io/ADC-12b-100M/manifests/design-top-adc_top.json`；
  - `rich-netlist/rnl` 的 Release 列表；
  - 从 GitHub API、raw 和 jsDelivr 匿名抓取 ADC 网表。

  以上均为 2026-11-04 实测。
