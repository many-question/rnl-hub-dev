# IP 库框架规格：草案 v0.8

- 日期：2026-11-04
- 状态：讨论稿，尚未实施。第 11 节列出需要人决定的事
- 版本变化：
  - v0.8（2026-11-05）：全平台只用 MIT 和 CERN-OHL-P-2.0 两种许可证，原来用 CC-BY-4.0 的目标、结果、验证、证据和笔记都改用 MIT（10.2 节）；D12 已定，第一版只收 GitHub
  - v0.7（2026-11-05）：账号不写进文件，由 CI 从 PR 取，文件里只有可选的自由文本 `author`；结果和验证编号去掉账号（第 1、7 节）。tb 改用 CERN-OHL-P-2.0，只有定义文本和 schema 用 MIT；不存副本，CERN-OHL-P-2.0 的外部实现自动收录，其他只引用（第 10 节重写）；D13、D14 已定
  - v0.6（2026-11-05）：作者名可以不填，只建议格式，去掉 `model`、`operator` 两个可选字段（第 1 节）；放弃专利覆盖全部贡献，定义和 tb 也写 `patents: waived`（10.3 节）；D18 已定
  - v0.5（2026-11-05）：收录的实现改用 CERN-OHL-P-2.0，每个收录的实现写 `patents: waived`，贡献说明着重强调放弃专利（10.2、10.3 节）；去掉参与者登记，作者署名见第 1 节；D16、D17 已定，新增 D18
  - v0.4（2026-11-05）：第 10 节加上专利（10.3 节）和"收录与副本"的区分（10.4 节），后面各小节顺延；D14、D15 已定，D13、D16、D17 改写
  - v0.3（2026-11-05）：按人类研究员定下的约定改写第 10 节。只引用实现，IP 定义一律在平台上；定义用 MIT，平台内的设计用 CC-BY-4.0，引用的实现按原许可证，宽松许可证的自动存副本。删除 v0.2 的"补存"；D9、D10 已定，新增 D13–D17
  - v0.2（2026-11-04）：加入第 10 节外部 IP（人类研究员提出），原第 10、11 节顺延为第 11、12 节
- 范围：把 [64 号文档](64_multi_agent_collaboration.md) v0.4 的设计落到可以实现的规格：
  - IP 定义的数据格式和目录结构；
  - tb 和实现的格式，以及网表怎样装配成仿真；
  - 结果和验证的目录结构；
  - PR 和 CI 的流程；
  - 网站怎样构建；
  - 外部实现怎样引用，以及许可证和专利。
- 关联：64 号文档定原则，本文定格式；两者冲突时以 64 号文档的已定约定为准。本文对 64 号文档的改动列在第 12 节
- 参考：
  - many-question/ADC-12b-100M 的 `docs/verification.md`、`CONTRIBUTING.md`、`.github/workflows/`，以及仓库目录（2026-11-03/04 读取）；
  - RNL 网页查看器的 `manifest.js`（清单格式 1，2026-11-04 读取）

## 0. 结论先行

1. **所有手写文件用 YAML，生成的文件用 JSON。** 每个文件第一行写 `schema: mp/<类型>/<n>`。对象合并后永不修改，所以格式只能出新版本，工具要永远能读旧版本。
2. **IP 名决定网表里的名字。** `seg-dac` 的子电路叫 `seg_dac`，端口顺序就是定义里 `pins` 的顺序。实现内部的辅助子电路一律以 `seg_dac__` 开头。
3. **测量是声明式的。** tb 只能调用平台维护的测量函数库，不能提交测量脚本。参与者会在自己机器上运行别人写的 tb，CI 也不能执行 PR 里的任何代码。
4. **一次仿真里，同一个 IP 只能用一个版本**（第一版的限制）。组合网表不写 include，由 `mpflow` 按固定的版本把文件装配成仿真；网站用同样的规则生成查看器清单。
5. **结果自带 `lock.json`**，记下实际用到的每个文件和它的 SHA-256。CI 按地址重算一遍，不一致就拒绝。验证必须针对同一个 lock。
6. **验证放在镜像结果路径的目录里**：`results/<月>/<结果>/` 对应 `verifications/<月>/<结果>/<验证>/`。每个对象只属于一个作者，没有人往别人的目录里加文件。
7. **一个 PR 是一个对象，或者一个"包"。** 包有三种：实现包（实现 + 它的结果）、新 IP 包（定义 + tb + 模型 + 模型上的结果）、组合包（组合 + 它新建的 IP 包 + 组合上的结果）。CI 从路径自动认出包。
8. **CI 用 `main` 上的工具检查 PR，只把 PR 当数据读。** 通过后由机器人放进合并队列，以 squash 方式合并。一个 PR 对应 `main` 上的一个提交，这个提交号就是对象的固定版本。
9. **网站增量构建。** 每次只处理上次构建以来新增的对象，输出按 IP 分片的索引、每个实现版本的查看器清单和订阅源。网表、证据按各自的合并提交号从 jsDelivr 现场抓取。
10. **外部只引用实现；许可证按路径划分；全部贡献放弃专利。** 定义文本和 `tools/` 用 MIT；实现和 tb 用 CERN-OHL-P-2.0，它自带专利许可；其余全部用 MIT，全平台只有这两种许可证。外部实现是 CERN-OHL-P-2.0 的自动收录，其他的只引用，平台记下提交号和每个文件的 SHA-256。结果和验证必须是平台内的原生提交。

## 1. 总原则

- **只有一个事实来源：`main` 上的文件。** 证据等级、可信度、预算兑现、排行都由工具从文件算出，不写进仓库，也不让作者自报。
- **作者写的少，工具算的多。** 能从别处算出来的字段不让作者写；必须写的（例如 lock），CI 重算核对。
- **格式版本：**
  - `schema: mp/ip/1` 这样的版本号写在每个手写文件里；
  - 格式改动出新版本（`mp/ip/2`），旧文件不迁移；
  - 工具、CI 和网站对每个版本都要能读，并在 `tools/schema/` 里保留全部版本的 JSON Schema。
- **YAML 子集：** UTF-8、LF 换行；只用安全加载能读的内容，不用锚点、别名和自定义标签。字段名用 ASCII，说明文字可以是中文或英文。
- **名字：**
  - IP 名：小写字母、数字和单个连字符，字母开头，不超过 40 个字符（64 号文档 4.3 节）；
  - 实现组名、tb 名、工况名：小写字母、数字、连字符和下划线；
  - 指标名：字母开头，字母、数字、下划线，不超过 32 个字符。它也是 VA 模型的参数名，所以要是合法的 Verilog-A 标识符；
  - 名字里的双下划线 `__` 留给工具生成的名字。
- **作者署名：**
  - **账号不写进文件。** CI 从 PR 取作者账号。网站构建时按合并提交查出对应的 PR 和作者账号（GitHub 接口 `GET /repos/<repo>/commits/<提交号>/pulls`），记进索引；这张对照表随增量状态保留，全量重建时不重查。作者检查、验证的独立性和可信度都按这个账号算。
  - **文件里只有一个可选的 `author` 字段，内容是自由文本**，真人和 agent 都可以署名。贡献说明建议 agent 写成 `<agent 名>(<模型名>)`，例如 `r08(fable51)`，真人写自己的名字；都不强制，因为有人不想公开所用的模型。署名限一行、不超过 64 个字符，不核对；可信度只认账号。
  - 网站同时显示署名和账号，例如"r08(fable51) · alice"；没署名时只显示账号。
  - 不需要参与者登记，参与者页由网站按账号汇总。

## 2. 仓库布局

```text
design/<IP>/                       一个 IP 的全部内容（第 3–7 节）
processes/<工艺名>.yaml            工艺登记：角名、器件名（3.4 节）
notes/<id>.md                      不属于某个 IP 的笔记
tools/                             mpflow、检查器、测量函数库、网站生成器、JSON Schema
.github/                           工作流、issue 表单、CODEOWNERS
CHARTER.md  CONTRIBUTING.md  README.md
LICENSE  LICENSES/                 许可证说明和全文；路径规则见 10.2 节
```

一个 IP 的目录：

```text
design/seg-dac/
  def/
    def.yaml                       定义（第 3 节）
    metrics/<序号>-<指标>.yaml      追加的指标
    tb/<tb>/v<n>/                  tb 版本（第 4 节）
  values/<id>.yaml                 数值集（第 6 节）
  impl/<实现组>/v<n>/              实现版本（第 5 节）
  results/<yyyy-mm>/<结果>/        结果（第 7 节）
  verifications/<yyyy-mm>/<结果>/<验证>/
  records/<id>.yaml                状态记录（第 6 节）
  notes/<id>.md
```

- 路径就是地址：`seg-dac@r08-seg/v2` 在 `design/seg-dac/impl/r08-seg/v2/`；`mega-precision:tb_sine/v1` 在 `design/mega-precision/def/tb/tb_sine/v1/`。
- 根 IP `mega-precision` 的定义里多一个 `env.yaml`（参考仿真环境）和 `externals/`（片外 R、C 的标准模型），见 64 号文档 5.1 节。

## 3. IP 定义

### 3.1 `def.yaml`

```yaml
schema: mp/ip/1
ip: seg-dac
title: 18 位分段电流舵 DAC
does: 把 18 位数字码转换成差分电流，满量程 ±2 mA      # 写做什么，不写怎么做
author: r08(fable51)                 # 署名，自由文本，可不填；账号由 CI 从 PR 取
based_on: []                         # 从别的 IP 改出来时写 {ref: <IP>, relation: derived}
patents: waived                      # 放弃专利，见 CONTRIBUTING（10.3 节）
process: <工艺名>                     # 必须在 processes/ 里登记过；片外器件写 external
pins:                                # 顺序就是子电路的端口顺序
  - {name: d,    dir: in,  kind: digital, width: 18}   # 展开成 d_0 … d_17，d_0 是最低位
  - {name: clk,  dir: in,  kind: digital}
  - {name: iop,  dir: out, kind: analog}
  - {name: ion,  dir: out, kind: analog}
  - {name: avdd, kind: supply, nominal: 3.3}
  - {name: dvdd, kind: supply, nominal: 1.8}
  - {name: gnd,  kind: ground}
metrics:
  - {id: inl,     unit: LSB, better: lower,  tb: tb_static,  tol: {abs: 0.05}, budget: true,
     desc: 端点拟合的积分非线性，取各码的最大绝对值}
  - {id: sfdr_1k, unit: dBc, better: higher, tb: tb_dynamic, tol: {abs: 0.5},  budget: true}
  - {id: power,   unit: mW,  better: lower,  tb: tb_dynamic, tol: {rel: 0.02}, budget: false}
```

| 字段 | 说明 |
|---|---|
| `pins` | 端口、电源和地都在这里，顺序固定。`kind` 取 `analog`、`digital`、`supply`、`ground`；`dir` 取 `in`、`out`、`inout`，电源和地不写 |
| `width` | 总线宽度。展开规则是 `<名>_<位>`，第 0 位是最低位。Spectre 没有总线，这样写不用转义 |
| `nominal` | 电源的标称电压。tb 按它和工况里的电源系数供电 |
| `metrics[].tb` | 用定义里的哪个 tb 测这个指标 |
| `metrics[].tol` | 复跑一致的容差：`abs`、`rel`，或两者取大。验证的一致性按它判定（7.4 节） |
| `metrics[].budget` | 是否是 VA 模型的参数。是的话，组合实现可以给它分配预算 |
| `better` | `lower`、`higher` 或 `none`。目标和预算按它判断是上限还是下限 |

根 IP 的电源组数不固定（WORK-007），所以根 IP 允许写 `supplies: open`：电源不列在 `pins` 里，由每个实现在 `impl.yaml` 里声明，tb 按声明供电（5.4 节）。其他 IP 的电源都要在 `pins` 里写死。

### 3.2 追加指标

- 只有定义者能追加，一次一个文件：`def/metrics/0001-hd3_1k.yaml`，内容和 `metrics` 里的一项相同。序号决定顺序。
- 追加的指标如果 `budget: true`，旧的模型实现里没有这个参数。网站把它显示为"模型未覆盖"，要等模型出新版本。

### 3.3 名字怎样落到网表

| 对象 | 网表里的名字 | 例子 |
|---|---|---|
| IP 的子电路 | IP 名，连字符换成下划线 | `seg_dac` |
| 实现内部的辅助子电路 | `<子电路>__<名>` | `seg_dac__unit_cell` |
| VA 模块 | `<子电路>__va` | `seg_dac__va` |
| 总线 | `<名>_<位>` | `d_0` |
| tb 里的被测实例 | 固定为 `XDUT` | |

IP 名只有单个连字符，换成下划线后不会出现 `__`，所以这些名字不会互相冲突，也不会和别的 IP 冲突。

### 3.4 工艺登记 `processes/<工艺名>.yaml`

```yaml
schema: mp/process/1
process: <工艺名>
corners: [tt, ss, ff, sf, fs]          # 标准角名，tb 只用这些名字
devices: [<器件名>, …]                 # 电路网表里允许出现的器件
```

- CI 没有 PDK，只能靠这个名单判断"只用了工艺器件"。
- 登记文件由人维护，属于受保护路径。
- 每个参与者在本地用一个绑定文件，把角名映射到自己的 PDK 安装，这个文件不提交（64 号文档 10.4 节）。
- **待确认：** 器件名算不算 PDK 信息（第 11 节 D6）。

### 3.5 接口指纹

网站用接口指纹提示疑似重复的 IP（64 号文档 4.3 节）。指纹由这些内容算出：工艺、各端口的 `kind`、`dir` 和 `width`（不含名字）、指标名的集合。

## 4. tb

### 4.1 目录

```text
design/seg-dac/def/tb/tb_static/v1/
  tb.yaml           说明：测哪些指标、工况、要保存的信号、测量方法
  tb.scs            tb 网表（Spectre + RNL）
  stimuli/          激励数据（可选，数据文件，体积计入上限）
  selftest.yaml     测量自测：用已知答案的理想被测对象检验测量本身
  README.md
```

### 4.2 `tb.yaml`

```yaml
schema: mp/tb/1
tb: tb_static
version: 1
author: r08(fable51)
patents: waived
measures: [inl, dnl]                   # 必须是定义里 tb 字段指向本 tb 的指标
netlist: tb.scs                        # 用规范名 seg_dac 实例化被测对象，实例名 XDUT
simulator: {name: spectre, min_version: "21.1"}
cases:
  tt_nom:      {corner: tt, temp: 27,  supply: 1.00}
  ss_low_125:  {corner: ss, temp: 125, supply: 0.95}
  ff_high_m40: {corner: ff, temp: -40, supply: 1.05}
required: [tt_nom]                     # 提交电路实现时至少要跑的工况
save: [XDUT.iop, XDUT.ion]             # 测量要用的信号
measure:                               # 只能调用 tools/measure 里的函数
  inl: {fn: static_linearity, args: {signal: "I(XDUT.iop) - I(XDUT.ion)", fit: endpoint}, take: inl_max}
  dnl: {fn: static_linearity, args: {signal: "I(XDUT.iop) - I(XDUT.ion)", fit: endpoint}, take: dnl_max}
evidence:                              # 每个工况保留哪些证据
  - {name: inl_curve, from: inl, take: inl_curve, kind: curve, max_points: 4096}
```

### 4.3 测量为什么必须是声明式

- 参与者要在自己机器上跑别人写的 tb 来做验证。如果 tb 能带 Python 脚本，验证别人的结果就等于在自己机器上运行陌生代码。
- CI 也不能执行 PR 里的代码（8.5 节）。
- 所以测量只能调用 `tools/measure` 里的函数：FFT 与谐波、积分噪声、静态线性度、建立时间、功耗统计等。函数库由人维护，属于受保护路径，有版本号。结果的 lock 里记下用的是哪个版本。
- 函数库不够用时，提 `platform` issue 申请新函数。新函数要带自测。
- 代价：测量的灵活性受限，新函数进库要等维护者。这是第 11 节 D1 要决定的事。

ADC-12b-100M 的测量写在 `tools/adcflow/measure/` 里，由负责人维护，参与者不写测量代码。这里的做法和它一致，只是函数要更通用，因为 tb 由各 IP 的定义者写。

### 4.4 tb 网表的约定

- 被测对象用规范子电路名实例化，实例名 `XDUT`，端口按定义的顺序连接。
- 电源按定义里的 `nominal` 乘以工况的 `supply` 系数给出。全局参数 `mp_supply`、`mp_temp` 由 `mpflow` 按工况写入。
- tb 网表里可以有理想元件和 tb 辅助的 VA 模块（放在 tb 目录里，名字以 `<IP 子电路>__tb_` 开头）。
- 工艺只写角名，由本地绑定文件映射到 PDK。
- 仿真精度设置写在 tb 网表里，属于 tb 的一部分。根 IP 的参考环境另见 `env.yaml`。

## 5. 实现

### 5.1 三种实现

| 种类 | 目录里有什么 | 网表规则 |
|---|---|---|
| `model` | `impl.yaml`、`model.scs`（规范子电路，内部是一个 VA 实例）、`<子电路>__va.va`、VA 模块的 RNL 符号包 `.rnl`、`README.md` | VA 参数覆盖定义里全部 `budget: true` 的指标，并带默认值 |
| `composition` | `impl.yaml`、组合网表、`README.md` | 只有 IP 实例和连线，没有器件 |
| `circuit` | `impl.yaml`、电路网表（可以分几个文件）、`README.md` | 工艺器件、本实现的辅助子电路；也可以有 IP 实例 |

- 电路实现也可以引用别的 IP，例如一个晶体管级 DAC 里用库里的运放 IP。这时它和组合一样要在 `instances` 里固定版本。它的证据等级按引用的版本递归计算（64 号文档 4.5 节）。
- 组合和电路的区别只在有没有器件。组合里没有器件，它的意义在于分解和预算。
- 数字实现的形式待定（64 号文档第 15 节第 5 项）。
- VA 模块的符号用 `.rnl` 包文件声明。ADC-12b-100M 的 `design/rnl/adc28_va.rnl` 就是这样做的。具体写法要用 `rnl check` 实测。

### 5.2 `impl.yaml`

模型：

```yaml
schema: mp/impl/1
kind: model
author: r08(fable51)
based_on: []
summary: 理想 DAC 加 INL 形状、热噪声和有限输出阻抗
patents: waived                        # 平台内的实现必须写，见 CONTRIBUTING（10.3 节）
files: {netlist: model.scs, va: [seg_dac__va.va], packages: [seg_dac__va.rnl]}
params: {inl: 0.3, sfdr_1k: 118}       # 默认值，也就是网站上的"模型"一列
ignores: [时钟馈通, 码相关的输出阻抗]    # 模型没有包含的非理想项
```

组合（64 号文档 4.6 节的例子，补上文件和电源）：

```yaml
schema: mp/impl/1
kind: composition
author: r07(fable51)
based_on: [{ref: mega-precision@r07-loop/v1, relation: revision}]
addresses: ["#42"]
summary: 数字合成主源 + 负载端检测闭环校正
patents: waived
files: {netlist: top.scs}
instances:
  dac:  {ip: seg-dac,      use: r08-seg/v2, budget: {inl: 0.5}}
  filt: {ip: recon-filter, use: model/v1,   budget: {hd3_1k: -140}}
  det:  {ip: err-detect,   use: model/v1,   budget: {spur_floor: -150}}
new_ips: [recon-filter]
supplies: {avdd5: 5.0, dvdd: 1.8}      # 只有根 IP（supplies: open）才写
```

电路：

```yaml
schema: mp/impl/1
kind: circuit
author: r08(fable51)
based_on: [{ref: seg-dac@r08-seg/v1, relation: revision}]
summary: 6+12 分段，动态元件匹配
patents: waived
files: {netlist: [seg_dac.scs, seg_dac__cells.scs]}
instances: {}                          # 引用别的 IP 时写，规则同组合
area: {value: 1.8, unit: mm2, basis: estimate}
```

### 5.3 网表规则

所有实现网表都要满足：

- Spectre 语法，带完整的 RNL 记录；`rnl check` 0 错误 0 警告。
- 不写 `include`、`ahdl_include`，不写绝对路径，不写工艺库语句（`include`/`section`）。
- 定义规范子电路，名字和端口顺序与定义一致。
- 只定义规范子电路和以 `<子电路>__` 开头的辅助子电路。
- 实例的主名称只能是：工艺登记里的器件（电路实现）、本实现的辅助子电路、`instances` 里固定的 IP 的规范子电路。
- 单个网表不超过 1 MB，整个实现包不超过 2 MB（64 号文档 6.1 节）。

预算不写进网表。装配时，如果某个实例固定到模型实现，`mpflow` 把预算作为参数加到这个实例上；固定到电路实现时不加，预算是否兑现看子 IP 的 tb 结果。

### 5.4 展开树与装配

被测对象加上替换，按固定的版本递归展开，得到一棵展开树。`mpflow run`、`mpflow verify`、CI 和网站用同一个展开函数：

1. 从被测的实现版本开始，按 `instances` 的 `use` 逐层展开；`override` 按实例路径（如 `dac.core`）替换。
2. **同一个 IP 在树里只能出现一个版本。** 出现两个版本时报错，并指出是哪两条路径，作者可以用 `override` 统一。原因是 Spectre 的子电路名是全局的，两个版本同名，不能并存。以后如果需要并存，由装配工具给子电路改名（第 11 节 D2）。
3. 收集文件：每个节点的网表、VA 文件和符号包。

`mpflow` 在本地把它们装配成仿真文件，这个文件不提交：

```text
tb.scs 的内容
+ 全局参数（工况）
+ 本地工艺绑定：角名 → PDK 的 include 和 section
+ 每个 VA 文件一行 ahdl_include
+ 展开树里每个网表一行 include（只在本地装配文件里出现）
+ 固定到模型的实例的预算参数
```

根 IP 的 `supplies: open` 在这里处理：tb 用一个标准接口，按实现声明的电源逐个加电压源，并逐域统计功耗。

## 6. 数值集与状态记录

### 6.1 数值集 `values/<id>.yaml`

```yaml
schema: mp/values/1
id: targets-v1
kind: targets                          # 目前只有目标；预算来自组合，模型数值来自模型实现
author: r08(fable51)
set:
  inl:     {max: 2.0}                  # 按指标的 better 写 max 或 min
  sfdr_1k: {min: 110, cases: [tt_nom]} # 不写 cases 就是全部必跑工况
```

- 只有定义者能写目标；根 IP 的目标由人写。
- 可以追加新的一套（`targets-v2`），旧的保留。网站默认显示最新的一套。

### 6.2 状态记录 `records/<id>.yaml`

```yaml
schema: mp/record/1
id: 20261201T0930Z-retract
kind: retract                          # retract 撤回 | erratum 勘误 | equivalent 等同 | claim 认领（10.10 节）
target: seg-dac@r08-seg/v1
author: r08(fable51)
text: 网表里 MSB 单元的宽长比写反了，结果无效；见 v2
```

- 撤回和勘误只能针对自己的对象。
- "等同"任何人都可以提，写成 `kind: equivalent, target: seg-dac, with: cs-dac-18b`。
- 网站把记录叠加在对象上显示，原对象不动。

## 7. 结果与验证

### 7.1 目录

```text
design/mega-precision/results/2026-11/20261120T0812Z-thd/
  result.yaml
  lock.json                          由 mpflow 生成
  evidence/f1k_r10k_tt.harmonics.csv
design/mega-precision/verifications/2026-11/20261120T0812Z-thd/20261122T1010Z/
  verification.yaml
  evidence/…                         可选
```

- 结果编号：`<UTC 时间>-<短名>`，按时间排序。编号里不放账号，因为账号不写进文件（第 1 节）。两个 PR 碰巧用了同一个编号时，合并队列会让后一个失败（路径已存在），重新生成编号即可。
- 月份目录取结果编号里的月份。验证放在镜像路径下，月份跟随被验证的结果，所以一个结果的全部验证在同一个目录里。
- 验证编号：`<UTC 时间>`。
- 每个结果、每个验证各自不超过 256 KB（64 号文档 6.1 节）。

### 7.2 `result.yaml`

```yaml
schema: mp/result/1
id: 20261120T0812Z-thd
author: r06
tb: mega-precision:tb_sine/v1
dut: mega-precision@r07-loop/v2
override: {det: err-detect@r05-het/v1}
status: ok                           # ok | partial（部分工况失败）| failed（负结果也收）
cases:
  f1k_r10k_tt:
    values: {THD: -133.2, POWER: 182.0}
    evidence: [evidence/f1k_r10k_tt.harmonics.csv]
  f1M_r10k_tt:
    values: {THD: -121.7, POWER: 190.5}
  f1M_r50_tt:
    error: 瞬态不收敛                 # 失败的工况写原因，不写数值
env: {simulator: spectre 23.1.0, process: <工艺名>, pdk_release: <发行名>, mpflow: 0.4.0}
cpu_hours: 6.5
note: 可选的一段说明
```

### 7.3 `lock.json`

```json
{
  "schema": "mp/lock/1",
  "tb":   {"ref": "mega-precision:tb_sine/v1", "files": {"tb.scs": "<sha256>"}},
  "tree": [
    {"path": "",    "ref": "mega-precision@r07-loop/v2", "files": {"top.scs": "<sha256>"}},
    {"path": "dac", "ref": "seg-dac@r08-seg/v2",         "files": {"seg_dac.scs": "<sha256>"}},
    {"path": "det", "ref": "err-detect@r05-het/v1",      "files": {"model.scs": "<sha256>", "err_detect__va.va": "<sha256>"}}
  ],
  "budgets": {"filt": {"hd3_1k": -140}},
  "measure": "tools/measure 1.2.0",
  "digest": "<以上内容的 SHA-256>"
}
```

- 地址确定了，文件就确定了，因为对象不可变。所以 CI 能按地址重算整个 lock，和提交的比较。不一致说明作者仿真时用的不是仓库里的文件，比如本地改过网表。
- lock 只记仓库里的东西。PDK 和仿真器只记名字和版本（`env`）。
- 这比 ADC 的依赖跟踪简单：ADC 的文件会被修改，要判断结果是否过期；这里文件不会变，不存在过期，只有"是否针对最新版本"。

### 7.4 验证与一致性

`verification.yaml`：

```yaml
schema: mp/verification/1
verifies: mega-precision!20261120T0812Z-thd
author: r02
lock_digest: <必须等于原结果 lock 的 digest>
cases:
  f1k_r10k_tt: {values: {THD: -133.1, POWER: 181.6}}
env: {simulator: spectre 23.1.0, process: <工艺名>, pdk_release: <发行名>, mpflow: 0.4.1}
cpu_hours: 3.1
```

- 验证者不写"一致"或"不一致"。网站按定义里每个指标的 `tol` 判定：|复跑 − 原值| ≤ max(abs, rel × |原值|)。
- 定义没写容差的指标，按单位用默认容差，例如 dB 类 ±0.5 dB、功耗 ±2%。默认表参照 ADC-12b-100M 的复核容差表，放在 `tools/` 里，由人维护。
- 一个结果的状态：
  - **自报**：没有验证；
  - **已验证**：有 n 个其他账号的验证，全部一致；
  - **有争议**：至少一个验证不一致。网站并列显示各方的数值和环境。
- `env` 里的 PDK 发行版或仿真器大版本和原结果不同时，网站标"跨环境"。跨环境的一致更有说服力，不一致也更可能是环境造成的。

### 7.5 证据

- 只收数据文件：`.csv` 和 `.json`。图由网站画。
- 证据的种类写在 tb 的 `evidence` 里：

| 种类 | 格式 | 网站怎样显示 |
|---|---|---|
| `spectrum` | CSV：`freq_Hz`，幅度列（如 `mag_dBc`） | 频谱图，标出谐波 |
| `curve` | CSV：第一列是横轴，其余是纵轴；表头写单位 | 曲线 |
| `waveform` | CSV：`t_s`，各信号列 | 波形，点数受 `max_points` 限制 |
| `table` | CSV | 表格 |
| `json` | JSON | 折叠的树 |

- 表头带单位后缀，例如 `freq_Hz`、`mag_dBc`、`i_A`。

## 8. PR 与 CI

### 8.1 PR 的形状

CI 从新增文件的路径认出对象，再认出 PR 的形状。一个 PR 只能是下面一种：

| 形状 | 内容 |
|---|---|
| 单个对象 | 一个结果、一个验证、一条状态记录、一篇笔记、一个追加的指标、一个 tb 新版本、一套数值集 |
| 实现包 | 一个实现版本，加上在它上面跑的结果 |
| 新 IP 包 | 定义、tb v1、`model/v1`，加上 tb 在模型上的结果 |
| 组合包 | 一个组合实现版本、它在 `new_ips` 里列出的新 IP 包，加上组合上的结果 |

包里的结果，被测对象必须是同一个包里的实现。

### 8.2 参与者的流程

```text
fork 仓库，克隆到本地
mpflow new impl seg-dac --group r08-seg         从模板生成目录和 impl.yaml
（在自己的环境里设计、仿真）
mpflow run seg-dac:tb_static --dut seg-dac@r08-seg/v1 --cases required
                                                装配、仿真、测量，生成结果目录和 lock
mpflow check                                    在本地跑 CI 的全部检查
git push 到自己 fork 的分支
mpflow pr                                       开 PR：标题以地址开头，默认是草稿
在 GitHub 上把 PR 标为就绪                         CI 通过后自动进入合并队列
```

### 8.3 工作流

| 工作流 | 触发 | 权限 | 做什么 |
|---|---|---|---|
| `check` | `pull_request`、`merge_group` | 只读 | 用 `main` 上的检查器检查 PR 的文件（8.4 节），输出报告 `report.json` 和摘要 |
| `report` | `check` 完成（`workflow_run`） | App | 把报告作为一条置顶评论贴到 PR 上；报告只当数据读 |
| `triage` | `pull_request_target`、`issues` | App | 打 `ip/<IP>`、`kind/*` 标签；校验 issue 地址，@ 相关作者；触发网站构建。不检出 PR 的代码 |
| `merge` | `check` 完成、每 10 分钟 | App | 对就绪、检查通过、没碰受保护路径的 PR 开启自动合并，进入合并队列 |
| `site` | push 到 `main`、`triage` 触发、每小时 | Pages | 增量构建网站（第 9 节）。单一并发组 |
| `audit` | 每天 | 只读 | 全库重新检查一遍，检查 jsDelivr 能否取到新合并的文件；问题写进网站，不让 `main` 变红 |

### 8.4 `check` 的检查项

按顺序执行。前面的步骤失败时，后面的仍然跑，报告列出全部问题：

| # | 检查 | 说明 |
|---|---|---|
| 1 | 只新增 | 没有修改、删除、改名；路径在允许的目录下 |
| 2 | 形状 | 认出对象和包（8.1 节） |
| 3 | 作者 | 账号取自 PR，已有对象的账号从合并历史查（第 1 节）；改进版只能由组作者（提交 v1 的账号）提交；指标、tb、目标只能由定义者提交；验证不能由原结果的账号提交；定义、tb 和收录的实现必须写 `patents: waived` |
| 4 | 格式 | 按 `schema` 字段做 JSON Schema 校验 |
| 5 | 名字与地址 | 路径和文件里的名字、地址一致；版本号连续，`v<n+1>` 之前 `v<n>` 已存在；编号里的时间和月份目录一致 |
| 6 | 引用 | 所有引用指向 `main` 上或本包里已有的对象，并且都是具体版本；`override` 的实例路径存在；展开树里同一个 IP 只有一个版本 |
| 7 | 网表 | `rnl check` 0 错误 0 警告；语句白名单（5.3 节）；规范子电路的名字和端口与定义一致；组合实例的端口和子 IP 定义一致 |
| 8 | 模型 | VA 参数覆盖全部 `budget: true` 的指标；用 OpenVAF 编译检查（可选，待定） |
| 9 | 结果 | 按地址重算 lock 并比较；工况在 tb 版本里存在；指标由这个 tb 测；实现包里有 `required` 的全部工况 |
| 10 | 验证 | 被验证的结果存在；`lock_digest` 等于原结果的；工况是原结果工况的子集 |
| 11 | 体积与卫生 | 文件类型白名单，不收二进制；大小上限；没有密钥样式的字符串；没有 PDK 内容（模型卡、库路径、`section=`）；UTF-8、LF |
| 12 | 受保护路径 | 标出 PR 是否碰了受保护路径，供合并机器人判断 |
| 13 | 来源与许可证 | 外部实现：地址合法、提交在声明的仓库里、文件哈希一致、许可证和声明一致；CERN-OHL-P-2.0 的必须带上文件，其他的不能带（10.7 节） |

报告里还带上标签建议，供 `triage` 使用。

### 8.5 安全

- **不执行 PR 里的任何东西。** `check` 用 `main` 上的检查器，PR 的文件只当数据读：YAML 安全加载，网表交给 RNL 解析，不运行仿真，不运行脚本。测量是声明式的（4.3 节），所以 PR 里也没有需要运行的脚本。
- 用 App 身份的工作流（`report`、`triage`、`merge`）不检出 PR 的代码。它们只读 GitHub API 返回的文件列表和 `check` 的报告，报告先校验格式和大小再用。
- fork 来的 PR 拿不到任何密钥。首次贡献的审批设为"仅限 GitHub 新用户"（65 号文档）。
- 受保护路径的改动必须由人批准：`.github/`、`tools/`、`processes/`、`CHARTER.md`、`CONTRIBUTING.md`、`LICENSE`、`LICENSES/`、根 IP 的 `def/` 和 `values/`。只要碰了，`check` 就不算数，合并机器人也不处理。

### 8.6 合并

- `main` 的规则集：
  - 必须通过 PR 合并；
  - 必须通过 `check`，来源限定为 GitHub Actions；
  - 必须走合并队列，合并方式为 squash；
  - 受保护路径要代码所有者批准；
  - 禁止强推和删除。
- 合并队列只在组织的公开仓库可用，本平台正好是（65 号文档）。它在合并前把 PR 和 `main` 的最新状态合起来再跑一次 `check`，所以两个 PR 互相引用、同时新建同名对象这类情况不会漏过。
- squash 合并让一个 PR 对应 `main` 上的一个提交。这个提交号就是 PR 里所有对象的"合并提交"，网站和查看器用它固定文件版本。
- 草稿 PR 不会进入队列。就绪但检查失败的 PR 留在原处，等作者推送修正。长期不动的 PR 由机器人先提醒、再关闭，期限待定。
- **待实测：** 规则集要求的批准数为 0 时，"代码所有者批准"是否仍然生效。不生效的话，受保护路径改由合并机器人拒绝处理、维护者手动合并。

## 9. 网站构建

### 9.1 输入与流程

`site` 工作流：

1. **读取上次的状态。** 从已发布的网站取 `index/state.json`：上次处理到的提交号，以及每个对象的合并提交号。
2. **找出新增的对象。** 用 `git log <上次>..HEAD --diff-filter=A --name-only` 列出新增文件，按路径认出对象，记下合并提交号。
3. **重算受影响的部分。** 新对象所在的 IP，以及引用了它的 IP（例如新结果影响被测实现和上层组合的预算兑现）。
4. **取 GitHub 的快照。** 用工作流自己的令牌查询开放的 issue 和 PR（标签、标题、草稿状态、PR 的 head 仓库和提交号）。
5. **输出并发布。** 用 `actions/deploy-pages` 发布。
6. **全量重建。** 工具版本变化或每周一次时全量重建，防止增量出错积累。

### 9.2 输出

| 路径 | 内容 | 大小量级 |
|---|---|---|
| `index/ips.json` | IP 列表：名字、标题、工艺、实现数、最好的证据等级、最近更新 | 每个 IP 约 300 B |
| `index/ip/<IP>.json` | 一个 IP 的全部：定义、数据表（目标、预算、模型、实测）、实现的版本树和合并提交号、结果和验证状态、预算兑现、"讨论"快照 | 每个 IP 几十到几百 KB |
| `index/participants.json`、`index/board.json` | 参与者统计（从各对象的作者字段汇总）；看板（待验证、有争议、长期无人回应） | |
| `manifests/<IP>/<组>/v<n>.json` | 每个实现版本的查看器清单 | 每个几 KB |
| `manifests/<IP>/results/<结果>.json` | 带 `override` 的结果的清单 | 同上 |
| `feeds/<IP>.atom`、`feeds/all.atom`、`feeds/new-ips.atom` | 订阅源（64 号文档 7.4 节） | 每个几 KB |
| `index/state.json` | 增量构建的状态 | |
| `app/`、`viewer/<版本>/` | 网页程序；固定版本的 RNL 查看器（64 号文档 8.4 节） | 约 4 MB |

### 9.3 查看器清单

RNL 查看器的清单格式 1 允许每个文件有自己的 `url`，并且必须写 SHA-256 和大小。所以：

- 清单列出展开树里每个网表和符号包，每个文件的 `url` 指向 jsDelivr 上它自己的合并提交，例如 `https://cdn.jsdelivr.net/gh/<组织>/<仓库>@<提交号>/design/seg-dac/impl/r08-seg/v2/seg_dac.scs`。
- 上层的组合网表不写 include，子电路靠清单的 `pool` 找到，这和 5.4 节的装配规则一致。
- 对象不可变，所以清单生成一次就永远有效。
- 查看器对清单的限制：最多 300 个文件，单个文件 2 MB，总共 20 MB。目前的体积上限都在范围内；很大的组合树会碰到 300 个文件的上限，到时再分层显示。
- 网页上"在查看器中打开"就是 `viewer/<版本>/?manifest=<本站>/manifests/....json`。

### 9.4 页面

单页程序，用 `#/` 开头的路由，所以不需要服务器端改写：

| 路由 | 页面 |
|---|---|
| `#/` | IP 库：按工艺、实现种类、证据等级筛选，搜索 |
| `#/ip/seg-dac` | IP 页：定义（端口表、指标表）、数据表、实现的版本树、结果列表、"讨论"栏、订阅源链接 |
| `#/ip/seg-dac@r08-seg/v2` | 实现页：说明、展开树、在查看器中打开、上面的结果、满足哪些组合的预算 |
| `#/ip/mega-precision!<结果>` | 结果页：各工况的数值、证据图、验证和一致性、lock 里的文件 |
| `#/board`、`#/participants/<账号>` | 看板、参与者 |

- 页面读本站的索引分片，再按需从 jsDelivr 取说明、笔记、证据和网表；jsDelivr 取不到时退到 `raw.githubusercontent.com`。不调用 GitHub API（65 号文档）。
- **进行中的设计（可选）：** 快照里有开放 PR 的 head 仓库和提交号。fork 是公开仓库，jsDelivr 也能取到，所以网站可以在查看器里显示还没合并的设计，并明确标为"未合并、未检查"。

## 10. 外部实现、许可证与专利

### 10.1 已定的约定

人类研究员 2026-11-04/05 定下：

- 平台除了内部的设计，也收录或引用外部符合格式的设计。
- **引用的是某个 IP 的实现，不是 IP 定义。** IP 定义一律在平台上。
- **许可证：**
  - 全平台只用两种许可证：平台内的实现和 tb 用 CERN-OHL-P-2.0；其余全部用 MIT，包括定义文本、目标、结果、验证、证据、笔记、格式 schema 和 `tools/`；
  - 外部实现：同样是 CERN-OHL-P-2.0 的，直接自动收录；其他许可证的一律只引用，按原许可证，平台列出每个实现的许可证要求。不存副本。
- **专利：**
  - 贡献者放弃对其贡献的专利权，覆盖全部贡献。
  - 不愿放弃的，不进收录，只能留作引用，并声明专利信息。这样别人才能放心使用平台里的 IP。
  - 贡献说明里要着重强调这一点。
- **作者署名：**
  - 账号由 CI 从 PR 自动取，不写进文件。
  - 文件里只有一个可选的自由文本字段 `author`，真人和 agent 都可以署名。贡献说明建议 agent 写成 `<agent 名>(<模型名>)`，但不强制。
  - IP 的实现列表同时列出署名和账号。
  - 不需要参与者登记（第 1 节）。

本节其余内容是落实这些约定的方案，还没有讨论。

### 10.2 许可证按路径划分

| 内容 | 许可证 | 状态 |
|---|---|---|
| `def/def.yaml` 和追加的指标（定义文本） | MIT | 已定 |
| `tools/`，包括格式 schema（`tools/schema/`） | MIT | 已定 |
| `def/tb/`（tb） | CERN-OHL-P-2.0 | 已定 |
| 平台内的实现（`impl/` 下没有 `origin` 的），以及收录的外部实现 | CERN-OHL-P-2.0 | 已定 |
| `values/`（目标） | MIT | 已定 |
| 结果、验证、证据、状态记录、笔记 | MIT | 已定 |
| 只引用的外部实现 | 原出处的许可证；文件不在平台里 | 已定 |

- **许可证文件：**
  - 根目录 `LICENSE` 写明上面的路径规则，`LICENSES/` 里放 CERN-OHL-P-2.0 和 MIT 的全文。它们都是受保护路径。
  - MIT 全文里的版权行写"Copyright (c) 2026 mega-precision 贡献者"，每个对象的具体作者看平台页面。
  - 平台内对象的许可证由路径决定，文件里不用另写。
  - 外部实现的许可证写在 `impl.yaml` 的 `origin.license` 里。收录时，目录里同时放原许可证文件。
- **声明（Notice）：** CERN-OHL-P-2.0 第 3 条要求保留声明；改过的版本要加一条声明，写明修改的日期和内容。所以实现和 tb 都要：
  - `README.md` 开头有一段标准声明：署名、许可证、平台地址。没署名时写"<平台地址> 的贡献者"，提交账号见平台页面。改进版还要写"基于 <地址> 修改"、日期和修改说明。这段声明由 `mpflow` 生成，CI 核对。
  - 网表、tb 网表和 VA 文件第一行写 `SPDX-License-Identifier: CERN-OHL-P-2.0` 的注释，这样单独拷走的文件也带着许可证信息。
- **提交即授权：**
  - GitHub 服务条款 D.6 规定：向带有许可证声明的仓库添加内容，就按同样的条款授权（"inbound=outbound"）。
  - CERN-OHL-P-2.0 自带专利许可（第 6.1 条），所以提交实现和 tb，就按这个许可证给出了专利许可。
  - 另外，定义、tb 和平台内的实现还要各自明写一行 `patents: waived`（10.3 节）。

### 10.3 专利

**公开和专利是什么关系。** 以下是我对公开资料的理解，不是法律意见：

- **CC-BY-4.0（最初考虑的许可证）本身不涉及专利。** 它的第 2(b)(2) 条明确写着不授予专利和商标权，所以"按 CC-BY-4.0 公开"不等于放弃专利。MIT 也没有明示的专利条款。
- **公开会让设计成为现有技术。** 此后别人不能再就它申请专利。至于贡献者本人：
  - 在中国和欧洲专利局，自己主动公开之后基本不能再申请。两地只有很窄的 6 个月宽限期（指定的展览会、学术会议、被他人违背意愿泄露等），在网上发布不在其列。
  - 在美国，发明人自己公开之后还有 1 年宽限期，这一年里仍可以申请。
- **公开之前已经申请或已经授权的专利不受影响。** 贡献者完全可以先申请专利、再贡献，之后照样主张。
- 所以只靠"公开"达不到"别人可以放心使用"，要让贡献者明确地给出专利许可。

**落实办法（已定，11-05）：实现和 tb 用 CERN-OHL-P-2.0。**

- 它是 CERN 专为硬件写的宽松许可证，经 OSI 批准。条件和 CC-BY 相近：保留声明、注明修改。
- **第 6.1 条：** 每个许可方向所有人授予免费、永久、全球、不可撤销的专利许可。范围是它有权许可、并且被这个设计必然实施的专利权利要求，用途包括制造、使用、销售和进口。
- **第 6.2 条：** 对这个设计发起专利诉讼的人，失去许可。
- 严格说，这是"给所有人免费许可"，不是注销专利；对使用者来说效果一样。

**分三处落实：**

1. **许可证：** 按路径规则（10.2 节）。
2. **写明 `patents: waived`：** 定义（`def.yaml`）、tb（`tb.yaml`）和平台内的实现（`impl.yaml`）都要写这一行。
   - `mpflow` 生成这些文件时自动带上这一行和一句注释；CI 要求必须有。
   - 追加的指标、目标、结果和笔记不另写，由贡献说明覆盖。
   - 这是写进仓库、合并后不能改的明确表示，比 PR 里的勾选框可靠，因为 PR 描述在合并后还能编辑。
3. **贡献说明着重强调。** `CONTRIBUTING.md` 的第一节（草稿）：

> **你向平台提交的全部内容，都放弃专利权。**
>
> - 这适用于你提交的一切：实现、tb、定义、目标、结果和笔记。你引用的外部实现不在此列，它的专利看原许可证和 `origin.patents`。
> - 实现和 tb 按 CERN-OHL-P-2.0 授权。按它的第 6.1 条，你向所有人授予免费、永久、全球、不可撤销的专利许可，覆盖你有权许可、并且被这个设计必然实施的全部专利权利要求。
> - 其余内容（定义、目标、结果和笔记）按 MIT 授权，MIT 没有专利条款。对这些内容，你另外向所有人授予同样的专利许可，范围和终止条件与 CERN-OHL-P-2.0 第 6.1、6.2 条相同。
> - 使用者不需要再向你取得专利许可。
> - 如果你或你的单位已经、或者打算为某个设计申请专利，并且想保留专利权：**不要提交收录。** 把它放在你自己的仓库里，作为外部实现引用，并在 `origin.patents` 里写明专利信息。
> - 提交前，请确认你有权这样做。设计如果属于职务发明，或者归你的单位所有，先取得单位的书面同意。
> - 平台不做专利检索，不保证收录的设计不侵犯第三方的专利。

PR 模板里也放一条要勾选的确认，内容同上。它只起提醒作用，记录以 `patents: waived` 为准。

草稿的第三条是平台自己写的承诺，超出了 MIT 本身的条款。它借用 CERN-OHL-P-2.0 的范围和终止条件，免得另起炉灶；措辞最好请懂专利的人看一遍。

**承诺管不到的：**

- **第三方的专利。** 网站要写明"不保证不侵犯第三方专利"。
- **贡献者无权处分的专利。** 只能靠贡献说明要求贡献者事先取得同意。

**想保留专利的设计**只能作为外部实现引用，并在 `origin.patents` 里写明已知的专利（10.6 节），网站照列（10.9 节）。

### 10.4 外部实现：收录或引用

| 原出处的许可证 | 怎样进平台 | 平台里放什么 | 装配和查看器用哪份 |
|---|---|---|---|
| CERN-OHL-P-2.0 | 自动收录 | `impl.yaml`、原样文件、原许可证文件 | 平台里的文件；外部仓库只是出处 |
| 其他开放许可证，例如 MIT、CC-BY-4.0、Apache-2.0、GPL、CERN-OHL-W/S | 只引用 | `impl.yaml` 和 `README.md`；`origin.files` 列出每个文件的 SHA-256 | 外部仓库，按提交号取 |
| 没有许可证 | 不收 | — | — |

- **为什么只收录 CERN-OHL-P-2.0。** 它和平台内实现的许可证相同，自带专利许可，收进来和平台自己的设计没有区别。其他许可证要么没有专利许可（MIT、CC-BY），要么条件和平台不同（Apache-2.0 的 NOTICE 文件要求，GPL 和 CERN-OHL-W/S 的互惠条款）。只引用它们，平台里存的实现就只有一种许可证。
- **"自动收录"由工具完成，不由机器人写仓库。**
  - `mpflow import` 按许可证决定带不带文件。
  - CI 强制：CERN-OHL-P-2.0 的必须带上文件和许可证文件，其他的不能带。
  - 这样收录和登记落在同一个合并提交里，不需要机器人另外提交，也不占内容创建配额。
- **没有许可证的仓库不收。** 按 GitHub 的说明，没有许可证时适用默认版权，别人不能复制、发布或改编。例如 ADC-12b-100M 目前没有许可证：2026-11-04 查 GitHub 的许可证接口，返回 404。
- **外部设计不符合格式时：**
  - CERN-OHL-P-2.0 的可以改编：改好后作为平台内的实现提交，保留原声明，并按第 3.3 条加一条修改声明。这就是一次普通的改进，`based_on` 写出处。
  - 其他许可证的不能改编，也不能引用，因为引用要求文件本来就符合格式。只能请原作者在自己的仓库里改好，再来引用。

### 10.5 外部实现挂在平台的定义下

- **必须实现平台上的某个 IP 定义。** 定义还不存在时，在同一个 PR 里新建，也就是一个新 IP 包：定义、tb、`model/v1`，其中定义和 tb 由提交者在平台上写。
- **外部仓库可以复制定义和 tb。** 定义是 MIT，可以随意复制；tb 是 CERN-OHL-P-2.0，复制时保留声明即可。外部作者可以对着它们开发。
- **外部实现守同样的规则：**
  - 规范子电路名和端口顺序（3.3 节）；
  - 网表规则（5.3 节）；
  - 体积上限，不含 PDK 内容；
  - 工艺已经登记。
- **结果和验证不收外部的。** 它们必须是平台内的原生提交，因为可信度按账号计算，代别人提交会把别人的工作记到自己账号下。外部作者可以作为参与者自己提交。
- **规则 7 不变。** R01–R10 的工作由各自的 agent 提交，别人不能把它当作外部实现引用或收录。
- **不能绕过平台的定义直接引用别的库里的实现。** Spectre 的子电路名是全局的，只有挂在平台的定义下，名字才不会冲突。

### 10.6 格式

外部实现的 `impl.yaml` 多一个 `origin`：

```yaml
schema: mp/impl/1
kind: circuit
author: r08(fable51)                 # 登记者的署名；原作者写在 origin.authors
origin:
  repo: <所有者>/<仓库>                # 第一版只收 GitHub 公开仓库
  commit: <40 位提交号>
  tag: v1.2.0                        # 可选，只用于显示
  path: ips/seg-dac/                 # 外部仓库里的目录
  license: MIT                       # SPDX 标识；CERN-OHL-P-2.0 自动收录，其他只引用
  license_file: LICENSE              # 外部仓库里的许可证文件；收录时同名保存
  authors: [<原作者，照外部仓库写>]
  patents: unknown                   # unknown 未声明 | none 声明没有 | 专利列表，例如
                                     # [{id: <专利或申请号>, holder: <权利人>, note: <覆盖什么>}]
  files:
    seg_dac.scs:        {sha256: <…>, size: 48120}
    seg_dac__cells.scs: {sha256: <…>, size: 20410}
summary: 6+12 分段电流舵
files: {netlist: [seg_dac.scs, seg_dac__cells.scs]}
```

- **目录内容：** 收录时，这些文件和许可证文件原样放在实现目录里；只引用时，目录里只有 `impl.yaml` 和 `README.md`。
- **专利：** 外部实现不写 `patents: waived`，因为登记者不能替原作者放弃专利。收录的 CERN-OHL-P-2.0 实现，专利许可来自许可证第 6.1 条；只引用的，专利看 `origin.patents`。`origin.patents` 由登记者照外部仓库的说明填，外部仓库没说的写 `unknown`。
- **地址：** 仍是平台的 `<IP>@<组>/v<n>`。外部仓库自己的版本名只写在 `tag` 里，用于显示。
- **lock：** `lock.json` 不变。它按地址记下每个文件的 SHA-256，不记文件放在哪里。CI 重算 lock 时，只引用的文件从外部取。
- **本地：** `mpflow` 取外部文件时核对 SHA-256，并按 SHA-256 缓存在本地。

### 10.7 来源和许可证核对

提交号能确定内容，但不能确定内容来自谁。GitHub 上同一个 fork 网络里的提交，可以通过网络里任何一个仓库按哈希访问，包括上游仓库，GitHub 认为这是有意的设计（Truffle Security，2024）。所以有人可以在自己的 fork 里提交，再把它登记成"来自某个知名仓库"。

因此登记时，CI 依次检查：

1. **地址合法：** `repo`、`path` 和文件名只允许常规字符，不允许 `..` 和绝对路径。
2. **提交确实在声明的仓库里：** 用比较接口 `GET /repos/<repo>/compare/<提交号>...<默认分支或 tag>` 确认这个提交在声明的仓库自己的历史里，状态应为 `ahead` 或 `identical`。只在 fork 里的提交应该得到 `diverged`，这一点待实测。
3. **文件内容一致：** 通过 jsDelivr 按提交号取每个文件，边取边限制大小，再比较 SHA-256。收录时，PR 里的文件和许可证文件还要和外部的逐字节相同。经 jsDelivr 取还有一个好处，见 10.8 节。
4. **许可证：**
   - 用许可证接口 `GET /repos/<repo>/license?ref=<提交号>` 查。接口用 Licensee 识别根目录的许可证文件，返回 `spdx_id`。识别结果和声明一致，才判断：CERN-OHL-P-2.0 的必须带上文件，其他的不能带。
   - 以下情况一律标"待人确认"，合并机器人不处理：识别不出；和声明不一致；许可证文件在子目录里（接口不看子目录）。
   - `ref` 能否直接写提交号，待实测：文档只举了分支和 PR 的例子。

这几步都在 `check` 里做，只读数据，不执行任何东西，和 8.5 节一致。核对只在登记时做一次。登记之后，外部仓库即使被删除或改写历史，也不影响对已登记对象内容的判断，因为哈希已经记下了。

### 10.8 只引用的实现：可用性

- **收录的外部实现**和平台内的对象一样：平台在，文件就在。
- **只引用的实现**可能消失：外部仓库被删除、转为私有，或者强推后旧提交被回收。
- **jsDelivr 能兜住大部分情况。** 它把取过的 GitHub 文件存进永久存储，按提交号的地址"实际上永久"缓存，仓库删了也还能取到（jsDelivr 的 README）。不过这只对取过的文件有效，而且 jsDelivr 没有服务保证（65 号文档）。所以 10.7 节第 3 步特意经 jsDelivr 取一遍，让每个文件在登记时就进入它的永久存储。
- **每天检查。** `audit` 每天检查这些文件能否从 jsDelivr 和 raw 取到。取不到时，网站把这个实现标为"外部文件不可用"，引用了它的组合标为"依赖不可用的外部文件"。
- **平台不存它们的拷贝。** 按已定的约定，CERN-OHL-P-2.0 以外的一律只引用。
- **代价：** 只引用的实现一旦外部和 jsDelivr 都取不到，引用了它的组合就无法再装配和复跑。组合作者选用只引用的实现时，`mpflow` 和网站都要提示这个风险。

### 10.9 列出许可证和专利信息

每个实现有一个专利状态：

| 状态 | 含义 |
|---|---|
| 已放弃 | 平台内的实现：CERN-OHL-P-2.0 第 6.1 条，并写明 `patents: waived`；收录的外部实现：CERN-OHL-P-2.0 第 6.1 条 |
| 许可证授予 | 只引用的实现，原许可证自带专利许可，例如 Apache-2.0 |
| 未授予 | 只引用的实现，原许可证没有专利许可，作者也没有声明有专利 |
| 已声明专利 | `origin.patents` 列出了专利 |

- **怎样判断：** 哪些许可证自带专利许可，由 `tools/` 里一张人维护的表决定。
- **实现页：** 显示本实现的许可证和专利状态，以及展开树里出现的全部许可证和专利状态、各自所在的实例。例如："本实现 CERN-OHL-P-2.0，已放弃；展开树里还有一个只引用的 GPL-3.0-only 实现（`dac` → `seg-dac@ext-foo/v1`）"。
- **结果页：** 同样列出它的展开树（含 `override`）。
- **IP 库页：** 可以筛选"展开树全部是已放弃"。这就是"可以放心使用"的视图。
- **只列不判。** 网站只列出要求，不判断许可证之间是否兼容，也不保证不侵犯第三方专利。
- **署名：** 每个对象给一行建议的署名：对象地址、署名和账号、"mega-precision IP 库"、许可证和对象页的链接。MIT 要求拷贝里保留版权和许可声明，CERN-OHL-P-2.0 要求保留声明（10.2 节）。

### 10.10 更新与认领

- **外部修改不影响平台。** 每个平台版本固定一个提交号。外部出了新版，由组作者提交 `v<n+1>`，指向新的提交号。`mpflow import` 一次生成整个目录。
- **不做自动同步。** 机器人替人开 PR 要占用内容创建配额（每个身份每小时 500 次）。外部的新提交也未必符合格式，未必是作者想发布的版本。
- **不需要原作者同意。** 只要许可证允许，任何人都可以引用；CERN-OHL-P-2.0 的会自动收录。网站显示"由 <账号> 收录（或引用），原作者 <…>"。
- **认领：**
  - 原作者以后加入平台，可以提交一条 `kind: claim` 的状态记录。
  - 个人仓库由 CI 核对 PR 作者是 `origin.repo` 的所有者；组织仓库的认领由人确认。
  - 认领后，原作者和登记者一起作为这个实现组的组作者（D11）。
  - 原作者如果愿意放弃专利，可以把只引用的实现作为新版本收录：`v<n+1>`，内容可以和上一版相同，按 CERN-OHL-P-2.0 授权并写明 `patents: waived`。从这个版本起，它的专利状态是"已放弃"；已有的版本不变。

### 10.11 对网站和 CI 的其他改动

- **来路标注：** 实现页标出来路，并显示许可证、专利状态和原作者。来路有四种：平台内、`收录自 <仓库>@<短提交号>`、`引用 <仓库>@<短提交号>`、`改编自 …`。IP 库页可以按来路和专利状态筛选。
- **查看器清单：**
  - 收录的外部实现，指向平台里文件的合并提交，和平台内的实现一样。
  - 只引用的实现，`url` 指向 `https://cdn.jsdelivr.net/gh/<所有者>/<仓库>@<提交号>/<路径>`，同样带 SHA-256 和大小。清单格式 1 本来就允许每个文件有自己的地址（9.3 节），不用改。
- **PR 形状不变。** 收录或引用一个外部实现就是一个实现包，或者新 IP 包里的实现；包里的结果照样必须由提交者自己跑。
- **CI：**
  - `check` 加第 13 项"来源与许可证"（10.7 节）。
  - 作者检查（8.4 节第 3 项）：定义、tb 和平台内的实现必须写 `patents: waived`。
  - 卫生检查（第 11 项）：实现和 tb 的 README 带标准声明，网表、tb 网表和 VA 文件带 SPDX 行（10.2 节）。
  - `audit` 加上可用性检查（10.8 节）。
- **受保护路径：** 加上 `LICENSE` 和 `LICENSES/`。许可证表在 `tools/` 里，本来就受保护。

## 11. 需要决定的事

| # | 问题 | 选项 | 建议 |
|---|---|---|---|
| D1 | 测量怎么写 | 只能调用平台的测量函数库 / 允许提交 Python 测量脚本 | 只用函数库；验证者不用在自己机器上运行陌生代码，CI 也不用运行任何东西 |
| D2 | 同一个 IP 的两个版本能否出现在同一次仿真里 | 第一版不能，用 `override` 统一 / 一开始就由装配工具改名支持并存 | 第一版不能；改名要同时改 RNL 记录，等真有需要再做 |
| D3 | 验证放在哪里 | 镜像结果路径的 `verifications/` / 放进结果目录 | 镜像路径；每个目录只属于一个作者，CI 的作者检查简单 |
| D4 | 合并方式 | 合并队列 + squash / 机器人直接合并 | 合并队列 + squash；队列在合并前按最新的 `main` 重查，一个 PR 一个提交 |
| D5 | 结果编号 | `<UTC>-<短名>` / 64 号文档原来的 `<agent>-<UTC>-<短名>` | `<UTC>-<短名>`；账号不写进文件，署名是自由文本、可能重复；撞号由合并队列挡住 |
| D6 | 工艺登记里写不写器件名 | 写器件名，CI 能查"只用工艺器件" / 不写，这一项改由验证者把关 | 需要人判断器件名是否属于 PDK 保密信息 |
| D7 | 电路实现能否引用别的 IP | 能，像组合一样固定版本 / 不能，必须拆成组合 | 能；晶体管级设计复用库里的模块是常态 |
| D8 | 网站增量状态放在哪里 | 已发布网站的 `index/state.json` / Actions 缓存 | 网站上；它本来就公开，缓存会过期 |
| D9 | 是否支持"引用"（文件留在外部） | — | **已定**（11-05）：引用的是实现；CERN-OHL-P-2.0 的自动收录，其他只引用（10.4 节） |
| D10 | 平台的许可证 | — | **已定**（11-05）：全平台只用两种：实现和 tb 用 CERN-OHL-P-2.0，其余全部用 MIT；引用的实现按原许可证；全部贡献放弃专利（10.1 节） |
| D11 | 原作者认领后的权限 | 和登记者一起作为组作者 / 转给原作者 | 一起；登记者已经在上面做了工作（结果、后续版本），不宜收回 |
| D12 | 外部来源的范围 | — | **已定**（11-05）：第一版只收 GitHub 公开仓库 |
| D13 | 外部实现存不存副本 | — | **已定**（11-05）：不存副本；CERN-OHL-P-2.0 的自动收录，其他只引用（10.4 节） |
| D14 | tb 和目标用什么许可证 | — | **已定**（11-05）：tb 用 CERN-OHL-P-2.0，目标用 MIT |
| D15 | `tools/` 用什么许可证 | — | **已定**（11-05）：MIT |
| D16 | 参与者要不要登记 | — | **已定**（11-05）：不登记。账号由 CI 从 PR 取，不写进文件；文件里只有可选的自由文本 `author`，真人和 agent 都能署名（第 1 节） |
| D17 | 放弃专利怎样落实 | — | **已定**（11-05）：实现和 tb 用 CERN-OHL-P-2.0；定义、tb 和平台内的实现写 `patents: waived`；贡献说明着重强调（10.3 节） |
| D18 | 放弃专利的声明是否也覆盖 MIT 的定义、tb 和目标 | — | **已定**（11-05）：覆盖全部贡献；定义、tb 和收录的实现各写 `patents: waived`（10.3 节） |

## 12. 对 64 号文档的改动

如果本文的建议被采纳，64 号文档要同步这些地方：

- 4.6 节：
  - 结果编号格式改为 `<UTC>-<短名>`，账号不写进文件；
  - 验证路径改为镜像结果路径，每个验证一个目录；
  - 结果加上 `lock.json`，`inputs_hash` 改为 `lock_digest`；
  - 加上 `processes/` 目录。
- 4.2 节：电路实现可以引用别的 IP。
- 6.1 节：PR 的形状改为"单个对象或三种包"（本文 8.1 节）。
- 8.1 节：CI 检查项以本文 8.4 节为准；合并方式改为合并队列 + squash。
- 第 1 节：外部实现和许可证的约定已记入 64 号文档 v0.5 的 1.1 节。
- 第 4 节：实现加上 `origin` 和 `patents`；仓库根目录加上 `LICENSE`、`LICENSES/`，许可证按路径划分（本文 10.2 节）。
- 8.1 节：检查项加上"来源与许可证"（本文 8.4 节第 13 项）；受保护路径加上 `LICENSE`、`LICENSES/`。
- 去掉参与者登记：第 4 节的对象表、4.6 节的目录、7.3 节、10.1 节、第 15 节第 10 项；作者署名改按本文第 1 节。
- 第 15 节：加上本文 D1、D2、D6，以及 D11。

## 来源

- many-question/ADC-12b-100M：`docs/verification.md`（复核记录放在候选路径之外、按单位的复核容差表、依赖哈希）、`CONTRIBUTING.md`、`.github/workflows/`，以及仓库目录（`design/behav/`、`design/xtor/`、`design/rnl/adc28_va.rnl`、`results/`、`verification/`），2026-11-03/04 读取。
- RNL 网页查看器 `https://rich-netlist.github.io/rnl/viewer/manifest.js`（清单格式 1：`base` 或每个文件的 `url`、SHA-256 和大小、`pool`、上限 300 个文件 / 单个 2 MB / 总共 20 MB），2026-11-04 读取。
- [Available rules for rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets)（代码所有者批准、合并方式、限定状态检查的来源）。
- GitHub 合并队列、Pages、jsDelivr 的数据和来源：见 [65 号文档](65_github_capacity.md)。
- [jsDelivr README](https://github.com/jsdelivr/jsdelivr)（GitHub 文件存进永久 S3 存储，仓库删除后仍可取；提交号地址"实际上永久"缓存，分支 12 小时；GitHub 单个文件默认上限 20 MB），2026-11-04 读取。
- [Get the license for a repository](https://docs.github.com/en/rest/licenses/licenses?apiVersion=2022-11-28)（可选的 `ref` 参数；用 Licensee 识别；返回 `spdx_id`），2026-11-04 读取。同日用该接口查 many-question/ADC-12b-100M，返回 404。
- [Licensing a repository](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/licensing-a-repository)（没有许可证时适用默认版权，别人不能复制、发布或改编）。
- [CC BY 4.0 法律文本](https://creativecommons.org/licenses/by/4.0/legalcode.en)（第 2(b)(2) 条不授予专利和商标权；第 3(a) 条署名条件，可以用链接满足），2026-11-05 读取。
- [Creative Commons FAQ](https://creativecommons.org/faq/)（不建议软件和硬件使用 CC 许可证；理由是没有源码分发、专利条款，与主流软件许可证不兼容），2026-11-05 读取。
- [GitHub Terms of Service](https://docs.github.com/en/site-policy/github-terms/github-terms-of-service) D.6 "Contributions Under Repository License"（inbound=outbound），2026-11-05 读取。
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) 第 3 条（贡献者的专利许可，发起专利诉讼时终止），2026-11-05 读取。
- [CC0 1.0 法律文本](https://creativecommons.org/publicdomain/zero/1.0/legalcode.en) 第 4(a) 条（不放弃、不许可专利和商标权），2026-11-05 读取。
- [CERN-OHL-P-2.0](https://choosealicense.com/licenses/cern-ohl-p-2.0/)（第 3 条保留声明、注明修改；第 6.1 条专利许可；第 6.2 条专利诉讼时终止），2026-11-05 读取。
- 专利宽限期：[Sterne Kessler: Patent Grace Period Laws in the IP5 Patent Offices](https://www.sternekessler.com/news-insights/insights/patent-grace-period-laws-ip5-patent-offices-some-similarities-largely/)、[patenttrademarkblog: patent grace periods](https://www.patenttrademarkblog.com/patent-grace-periods/)（二手资料：美国 35 U.S.C. §102(b)(1) 一年；中国专利法第 24 条、EPC 第 55 条只有限定情形下的六个月），2026-11-05 读取。
- Truffle Security, [Anyone can Access Deleted and Private Repository Data on GitHub](https://trufflesecurity.com/blog/anyone-can-access-deleted-and-private-repo-data-github)（2024-07；fork 网络里的提交可以经上游仓库按哈希访问，GitHub 回复说这是有意的设计）。
