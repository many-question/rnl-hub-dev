# 设计草稿

这三份草稿是 2026-11-03 至 11-05 在 Mega Precision 仓库里起草的（原编号 64、65、66），11-05 原样搬到这里。之后平台改名为 rnl-hub，和 Mega Precision 不再有关系。

| 文件 | 内容 | 版本 |
|---|---|---|
| [64_multi_agent_collaboration.md](64_multi_agent_collaboration.md) | 总体设计：对象模型、协作流程、issue 与 PR、网站 | v0.5 |
| [65_github_capacity.md](65_github_capacity.md) | GitHub 的承载能力、组织与套餐、网站托管 | v0.3 |
| [66_framework_spec.md](66_framework_spec.md) | 框架规格：文件格式、目录、PR 与 CI、网站构建、外部实现、许可证与专利 | v0.8 |

**这些草稿已部分过时，以 [../decisions.md](../decisions.md) 为准。** 主要的过时内容：

- 以 mega-precision 为根 IP、R01–R10 作为首批参与者；
- 统一仿真器；
- IP 定义里写工艺和指标（现在只有名字、描述和 IO），tb 只能由定义者写；
- 测量只能调用平台的函数库；
- "模型、组合、电路"三种实现（现在是实现类型和工艺两个字段），以及给 VA 子模块注入预算参数；
- 实现必须带结果；
- 结果和验证分开存放。

草稿不再修改，新的规格另写新文件。
